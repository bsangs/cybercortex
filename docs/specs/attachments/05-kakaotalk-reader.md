# 05 — kakaotalk-reader.swift

macOS AXUIElement API로 카카오톡 채팅방 메시지를 읽는 Swift 스크립트.

컴파일: `swiftc -o agent/scripts/kakaotalk-reader agent/scripts/kakaotalk-reader.swift`

**주의**: 카카오톡 AX 트리는 버전에 따라 달라질 수 있다. `ax-dump.lua`로 구조를 먼저 확인하고, 필요 시 extractMessages() 함수의 셀렉터를 조정해야 한다.

## agent/scripts/kakaotalk-reader.swift

```swift
#!/usr/bin/env swift
// kakaotalk-reader.swift
// Reads KakaoTalk chat messages via macOS Accessibility API
// Outputs raw markdown files to sources/raw/

import Cocoa
import Foundation
import CommonCrypto

// MARK: - Constants

let PROJECT_NAME = "{project}"  // ← 프로젝트 디렉토리명으로 변경
let baseDir = FileManager.default.homeDirectoryForCurrentUser.appendingPathComponent(PROJECT_NAME)
let rawDir = baseDir.appendingPathComponent("sources/raw")
let seenHashesPath = baseDir.appendingPathComponent("sources/.seen_hashes")

// MARK: - Dedup

var seenHashes = Set<String>()

func loadSeenHashes() {
    guard let content = try? String(contentsOf: seenHashesPath, encoding: .utf8) else { return }
    for line in content.split(separator: "\n") {
        let h = line.trimmingCharacters(in: .whitespaces)
        if !h.isEmpty { seenHashes.insert(h) }
    }
}

func hashMessage(_ sender: String, _ text: String, _ time: String) -> String {
    let key = "\(sender)\n\(text)\n\(time)"
    let data = Data(key.utf8)
    var digest = [UInt8](repeating: 0, count: Int(CC_SHA256_DIGEST_LENGTH))
    data.withUnsafeBytes { CC_SHA256($0.baseAddress, CC_LONG(data.count), &digest) }
    return digest.map { String(format: "%02x", $0) }.joined()
}

func appendHash(_ hash: String) {
    seenHashes.insert(hash)
    if let fh = FileHandle(forWritingAtPath: seenHashesPath.path) {
        fh.seekToEndOfFile()
        fh.write(Data("\(hash)\n".utf8))
        fh.closeFile()
    } else {
        try? "\(hash)\n".write(to: seenHashesPath, atomically: false, encoding: .utf8)
    }
}

func trimSeenHashes() {
    guard seenHashes.count > 10000 else { return }
    guard let content = try? String(contentsOf: seenHashesPath, encoding: .utf8) else { return }
    let lines = content.split(separator: "\n")
    let keep = lines.suffix(5000)
    seenHashes = Set(keep.map { String($0) })
    try? keep.joined(separator: "\n").appending("\n").write(to: seenHashesPath, atomically: true, encoding: .utf8)
}

// MARK: - AX Helpers

struct ChatMessage {
    let time: String
    let sender: String
    let text: String
}

func getAXValue(_ element: AXUIElement, _ attribute: String) -> AnyObject? {
    var value: AnyObject?
    AXUIElementCopyAttributeValue(element, attribute as CFString, &value)
    return value
}

func getStringValue(_ element: AXUIElement, _ attribute: String) -> String? {
    guard let value = getAXValue(element, attribute) else { return nil }
    return value as? String
}

func getChildren(_ element: AXUIElement) -> [AXUIElement] {
    guard let children = getAXValue(element, kAXChildrenAttribute) as? [AXUIElement] else { return [] }
    return children
}

func getFrame(_ element: AXUIElement) -> CGRect? {
    var ref: AnyObject?
    guard AXUIElementCopyAttributeValue(element, "AXFrame" as CFString, &ref) == .success else { return nil }
    var rect = CGRect.zero
    guard AXValueGetValue(ref as! AXValue, .cgRect, &rect) else { return nil }
    return rect
}

func getRole(_ element: AXUIElement) -> String {
    return getStringValue(element, kAXRoleAttribute) ?? ""
}

func getSubrole(_ element: AXUIElement) -> String {
    return getStringValue(element, kAXSubroleAttribute) ?? ""
}

// MARK: - Message Extraction

func extractMessages(from window: AXUIElement) -> [ChatMessage] {
    var messages = [ChatMessage]()

    let windowMidX: CGFloat = {
        if let wf = getFrame(window) { return wf.origin.x + wf.size.width / 2 }
        return 400
    }()

    let winChildren = getChildren(window)
    guard let scrollArea = winChildren.first(where: { getRole($0) == "AXScrollArea" }) else { return messages }
    let saChildren = getChildren(scrollArea)
    guard let table = saChildren.first(where: { getRole($0) == "AXTable" }) else { return messages }

    let rows = getChildren(table)
    let startIdx = max(0, rows.count - 30)

    var lastSender = ""
    var lastTime = ""

    for i in startIdx..<rows.count {
        let row = rows[i]
        guard getRole(row) == "AXRow" else { continue }

        let rowChildren = getChildren(row)
        guard let cell = rowChildren.first(where: { getRole($0) == "AXCell" }) else { continue }

        let cellChildren = getChildren(cell)

        var timeLabel: String?
        var senderName: String?
        var messageText: String?
        var textAreaElem: AXUIElement?
        var hasProfile = false
        var fileName: String?
        var fileSize: String?

        for elem in cellChildren {
            let role = getRole(elem)

            if role == "AXButton" {
                let desc = getStringValue(elem, kAXDescriptionAttribute) ?? ""
                if desc == "Profile" { hasProfile = true }
                continue
            } else if role == "AXStaticText" {
                var val = getStringValue(elem, kAXValueAttribute) ?? ""
                if val == "Edited" { continue }

                let mergedRegex = try! NSRegularExpression(pattern: "^\\d+\\n(\\d{1,2}:\\d{2})$")
                if let match = mergedRegex.firstMatch(in: val, range: NSRange(val.startIndex..., in: val)),
                   let timeRange = Range(match.range(at: 1), in: val) {
                    val = String(val[timeRange])
                }

                let pureNumberRegex = try! NSRegularExpression(pattern: "^\\d+$")
                if pureNumberRegex.firstMatch(in: val, range: NSRange(val.startIndex..., in: val)) != nil { continue }

                let timeRegex = try! NSRegularExpression(pattern: "^\\d{1,2}:\\d{2}$")
                if timeRegex.firstMatch(in: val, range: NSRange(val.startIndex..., in: val)) != nil {
                    timeLabel = val
                } else if val.hasPrefix("Size:") {
                    fileSize = val
                } else if val.hasPrefix("Expiry") || val == "· " {
                    continue
                } else if senderName == nil {
                    senderName = val
                } else if fileName == nil && senderName != nil {
                    fileName = val
                }
            } else if role == "AXTextArea" {
                let val = getStringValue(elem, kAXValueAttribute) ?? ""
                if !val.isEmpty { messageText = val; textAreaElem = elem }
            }
        }

        if messageText == nil, let fn = fileName {
            let sizeStr = fileSize.map { " (\($0.replacingOccurrences(of: "Size: ", with: "")))" } ?? ""
            messageText = "[파일] \(fn)\(sizeStr)"
        }

        guard let text = messageText else { continue }

        if let t = timeLabel { lastTime = t }

        let isRightAligned: Bool? = {
            guard let ta = textAreaElem, let f = getFrame(ta) else { return nil }
            return f.midX > windowMidX
        }()

        let effectiveSender: String
        if hasProfile, let s = senderName {
            lastSender = s; effectiveSender = s
        } else if let s = senderName {
            lastSender = s; effectiveSender = s
        } else if isRightAligned == true {
            lastSender = "나"; effectiveSender = "나"
        } else if isRightAligned == false {
            effectiveSender = lastSender
        } else {
            lastSender = "나"; effectiveSender = "나"
        }
        guard !effectiveSender.isEmpty else { continue }

        messages.append(ChatMessage(time: timeLabel ?? lastTime, sender: effectiveSender, text: text))
    }

    return messages
}

// MARK: - File Output

func safeRoomName(_ name: String) -> String {
    var result = ""
    for scalar in name.unicodeScalars {
        if scalar.isASCII {
            if scalar.value >= 0x30 && scalar.value <= 0x39
                || scalar.value >= 0x41 && scalar.value <= 0x5A
                || scalar.value >= 0x61 && scalar.value <= 0x7A {
                result.append(Character(scalar))
            } else { result.append("-") }
        } else { result.append(Character(scalar)) }
    }
    while result.contains("--") { result = result.replacingOccurrences(of: "--", with: "-") }
    result = result.trimmingCharacters(in: CharacterSet(charactersIn: "-"))
    return result.isEmpty ? "unknown" : result
}

func writeRawFile(roomName: String, messages: [ChatMessage]) {
    let formatter = DateFormatter()
    formatter.dateFormat = "yyyyMMdd-HHmmss"
    let ts = formatter.string(from: Date())

    let isoFormatter = DateFormatter()
    isoFormatter.dateFormat = "yyyy-MM-dd'T'HH:mm:ssXXXXX"
    isoFormatter.timeZone = TimeZone(identifier: "Asia/Seoul")
    let captured = isoFormatter.string(from: Date())

    let safeName = safeRoomName(roomName)
    let filename = "kakaotalk-\(safeName)-\(ts).md"
    let outPath = rawDir.appendingPathComponent(filename)

    var content = """
    ---
    source: kakaotalk
    room: \(roomName)
    captured: \(captured)
    ---

    
    """

    for msg in messages {
        content += "- [\(msg.time)] \(msg.sender): \(msg.text)\n"
    }

    try? content.write(to: outPath, atomically: true, encoding: .utf8)
    print("[kakaotalk-swift] Saved \(messages.count) messages → \(filename)")
}

// MARK: - Main

loadSeenHashes()

let app = NSWorkspace.shared.runningApplications.first { $0.localizedName == "KakaoTalk" }
guard let app = app else {
    print("[kakaotalk-swift] KakaoTalk not running")
    exit(0)
}

let axApp = AXUIElementCreateApplication(app.processIdentifier)

guard let windowsRef = getAXValue(axApp, kAXWindowsAttribute) as? [AXUIElement] else {
    print("[kakaotalk-swift] No windows found")
    exit(0)
}

let skipTitles: Set<String> = ["KakaoTalk", "카카오톡", ""]
var totalNew = 0

for window in windowsRef {
    let title = getStringValue(window, kAXTitleAttribute) ?? ""
    if skipTitles.contains(title) { continue }

    let messages = extractMessages(from: window)

    var newMessages = [ChatMessage]()
    for msg in messages {
        let hash = hashMessage(msg.sender, msg.text, msg.time)
        if !seenHashes.contains(hash) {
            newMessages.append(msg)
            appendHash(hash)
        }
    }

    if newMessages.isEmpty {
        print("[kakaotalk-swift] No new messages in '\(title)'")
    } else {
        writeRawFile(roomName: title, messages: newMessages)
        totalNew += newMessages.count
    }
}

trimSeenHashes()
print("[kakaotalk-swift] Done — \(totalNew) new messages total")
```
