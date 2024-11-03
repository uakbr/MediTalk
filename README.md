**MediTalk MVP**

---

Below is the full code for the **MediTalk** application MVP, adhering to best practices in app design, app language, and featuring a modern, intuitive UI/UX. The application integrates Supabase for user authentication and uses OpenAI's Realtime API for real-time voice translation. Please ensure to replace placeholder values like `"YOUR_SUPABASE_API_KEY"`, `"YOUR_SUPABASE_URL"`, and `"YOUR_OPENAI_API_KEY"` with your actual credentials when implementing the code.

---

### Project Structure

```
mediTalk/
├── README.md
├── LICENSE
├── .gitignore
├── MediTalk.xcodeproj
├── MediTalk/
│   ├── AppDelegate.swift
│   ├── SceneDelegate.swift
│   ├── ContentView.swift
│   ├── Models/
│   │   ├── User.swift
│   │   └── Conversation.swift
│   ├── Views/
│   │   ├── LoginView.swift
│   │   ├── SignUpView.swift
│   │   ├── MainView.swift
│   │   ├── ConversationView.swift
│   │   └── SettingsView.swift
│   ├── ViewModels/
│   │   ├── AuthViewModel.swift
│   │   └── ConversationViewModel.swift
│   ├── Services/
│   │   ├── RealtimeAPIService.swift
│   │   ├── AudioService.swift
│   │   └── AuthService.swift
│   ├── Resources/
│   │   ├── Assets.xcassets
│   │   └── Localizable.strings
│   └── Info.plist
├── Tests/
│   ├── MediTalkTests/
│   │   └── MediTalkTests.swift
│   └── MediTalkUITests/
│       └── MediTalkUITests.swift
└── Documentation/
    └── TechnicalSpecification.md
```

---

### Complete Code Files

#### 1. **AppDelegate.swift**

```swift
import UIKit
import SwiftUI

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    var window: UIWindow?
    var authViewModel = AuthViewModel()
    
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        let contentView = ContentView()
            .environmentObject(authViewModel)
        
        let window = UIWindow(frame: UIScreen.main.bounds)
        window.rootViewController = UIHostingController(rootView: contentView)
        self.window = window
        window.makeKeyAndVisible()
        
        return true
    }
}
```

#### 2. **ContentView.swift**

```swift
import SwiftUI

struct ContentView: View {
    @EnvironmentObject var authViewModel: AuthViewModel
    
    var body: some View {
        Group {
            if authViewModel.isAuthenticated {
                MainView()
                    .environmentObject(authViewModel)
            } else {
                LoginView()
                    .environmentObject(authViewModel)
            }
        }
    }
}
```

#### 3. **Models/User.swift**

```swift
import Foundation

struct User {
    let id: String
    let email: String
}
```

#### 4. **Models/Conversation.swift**

```swift
import Foundation

struct Conversation: Identifiable {
    let id: UUID
    let messages: [String]
    let timestamp: Date
}
```

#### 5. **Services/AuthService.swift**

```swift
import Foundation
import Supabase

class AuthService {
    static let shared = AuthService()
    private var client: SupabaseClient!
    
    private init() {
        let url = URL(string: "YOUR_SUPABASE_URL")!
        let apiKey = "YOUR_SUPABASE_API_KEY"
        client = SupabaseClient(supabaseURL: url, supabaseKey: apiKey)
    }
    
    func signUp(email: String, password: String, completion: @escaping (Result<User, Error>) -> Void) {
        client.auth.signUp(email: email, password: password) { result in
            switch result {
            case .success(let session):
                if let user = session.user {
                    completion(.success(User(id: user.id, email: user.email ?? "")))
                } else {
                    completion(.failure(NSError(domain: "AuthService", code: -1, userInfo: [NSLocalizedDescriptionKey: "No user data available."])))
                }
            case .failure(let error):
                completion(.failure(error))
            }
        }
    }
    
    func signIn(email: String, password: String, completion: @escaping (Result<User, Error>) -> Void) {
        client.auth.signIn(email: email, password: password) { result in
            switch result {
            case .success(let session):
                if let user = session.user {
                    completion(.success(User(id: user.id, email: user.email ?? "")))
                } else {
                    completion(.failure(NSError(domain: "AuthService", code: -1, userInfo: [NSLocalizedDescriptionKey: "No user data available."])))
                }
            case .failure(let error):
                completion(.failure(error))
            }
        }
    }
    
    func signOut(completion: @escaping (Result<Void, Error>) -> Void) {
        client.auth.signOut { result in
            completion(result)
        }
    }
    
    func getSessionUser() -> User? {
        if let user = client.auth.session?.user {
            return User(id: user.id, email: user.email ?? "")
        }
        return nil
    }
}
```

#### 6. **Services/RealtimeAPIService.swift**

```swift
import Foundation
import Starscream

class RealtimeAPIService: WebSocketDelegate {
    private var socket: WebSocket?
    private let apiKey = "YOUR_OPENAI_API_KEY"
    private let model = "gpt-4o-realtime-preview-2024-10-01"
    var isConnected: Bool = false
    
    // Callbacks
    var onReceiveAudio: ((Data) -> Void)?
    var onReceiveTranscription: ((String) -> Void)?
    var onError: ((Error) -> Void)?
    
    func connect() {
        var request = URLRequest(url: URL(string: "wss://api.openai.com/v1/realtime?model=\(model)")!)
        request.setValue("Bearer \(apiKey)", forHTTPHeaderField: "Authorization")
        request.setValue("realtime=v1", forHTTPHeaderField: "OpenAI-Beta")
        
        socket = WebSocket(request: request)
        socket?.delegate = self
        socket?.connect()
    }
    
    func disconnect() {
        socket?.disconnect()
    }
    
    func sendAudioData(_ data: Data) {
        guard isConnected else { return }
        
        let base64Audio = data.base64EncodedString()
        let event: [String: Any] = [
            "type": "input_audio_buffer.append",
            "audio": base64Audio,
            "audio_specification": [
                "type": "pcm_s16le",
                "sampling_rate": 24000,
                "channels": 1
            ]
        ]
        
        if let jsonData = try? JSONSerialization.data(withJSONObject: event) {
            socket?.write(data: jsonData)
        }
    }
    
    // MARK: - WebSocketDelegate Methods
    func didReceive(event: WebSocketEvent, client: WebSocket) {
        switch event {
        case .connected(_):
            isConnected = true
            print("WebSocket connected")
        case .disconnected(let reason, let code):
            isConnected = false
            print("WebSocket disconnected: \(reason) (\(code))")
        case .text(let string):
            handleMessage(string)
        case .error(let error):
            isConnected = false
            print("WebSocket error: \(String(describing: error))")
            onError?(error ?? NSError(domain: "WebSocket", code: -1, userInfo: nil))
        default:
            break
        }
    }
    
    private func handleMessage(_ message: String) {
        guard let data = message.data(using: .utf8) else { return }
        do {
            if let json = try JSONSerialization.jsonObject(with: data) as? [String: Any],
               let eventType = json["type"] as? String {
                switch eventType {
                case "response.audio.delta":
                    if let audioBase64 = json["audio"] as? String,
                       let audioData = Data(base64Encoded: audioBase64) {
                        onReceiveAudio?(audioData)
                    }
                case "response.text.delta":
                    if let text = json["text"] as? String {
                        onReceiveTranscription?(text)
                    }
                case "error":
                    if let errorInfo = json["error"] as? [String: Any],
                       let message = errorInfo["message"] as? String {
                        let error = NSError(domain: "RealtimeAPI", code: -1, userInfo: [NSLocalizedDescriptionKey: message])
                        onError?(error)
                    }
                default:
                    break
                }
            }
        } catch {
            print("Failed to parse message: \(error.localizedDescription)")
        }
    }
}
```

#### 7. **Services/AudioService.swift**

```swift
import Foundation
import AVFoundation

class AudioService: NSObject, AVAudioRecorderDelegate, AVAudioPlayerDelegate {
    private var audioRecorder: AVAudioRecorder?
    private var audioPlayer: AVAudioPlayer?
    var isRecording = false
    var onAudioDataAvailable: ((Data) -> Void)?
    
    func startRecording() {
        let settings: [String: Any] = [
            AVFormatIDKey: kAudioFormatLinearPCM,
            AVSampleRateKey: 24000,
            AVNumberOfChannelsKey: 1,
            AVLinearPCMBitDepthKey: 16,
            AVLinearPCMIsFloatKey: false,
            AVEncoderAudioQualityKey: AVAudioQuality.high.rawValue
        ]
        
        let tempDir = NSTemporaryDirectory()
        let filePath = tempDir + "/tempAudio.caf"
        let url = URL(fileURLWithPath: filePath)
        
        do {
            try AVAudioSession.sharedInstance().setCategory(.playAndRecord, options: [.defaultToSpeaker])
            try AVAudioSession.sharedInstance().setActive(true)
            audioRecorder = try AVAudioRecorder(url: url, settings: settings)
            audioRecorder?.delegate = self
            audioRecorder?.isMeteringEnabled = true
            audioRecorder?.record()
            isRecording = true
        } catch {
            print("Failed to start recording: \(error.localizedDescription)")
        }
    }
    
    func stopRecording() {
        audioRecorder?.stop()
        isRecording = false
    }
    
    func playAudioData(_ data: Data) {
        do {
            audioPlayer = try AVAudioPlayer(data: data)
            audioPlayer?.delegate = self
            audioPlayer?.prepareToPlay()
            audioPlayer?.play()
        } catch {
            print("Failed to play audio: \(error.localizedDescription)")
        }
    }
    
    // MARK: - AVAudioRecorderDelegate
    func audioRecorderDidFinishRecording(_ recorder: AVAudioRecorder, successfully flag: Bool) {
        if flag, let data = try? Data(contentsOf: recorder.url) {
            onAudioDataAvailable?(data)
        }
    }
    
    // MARK: - AVAudioPlayerDelegate
    func audioPlayerDidFinishPlaying(_ player: AVAudioPlayer, successfully flag: Bool) {
        // Handle completion if needed
    }
}
```

#### 8. **ViewModels/AuthViewModel.swift**

```swift
import Foundation
import Combine

class AuthViewModel: ObservableObject {
    @Published var isAuthenticated = false
    @Published var errorMessage: String?
    @Published var userEmail: String?
    
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        checkAuthentication()
    }
    
    func checkAuthentication() {
        if let user = AuthService.shared.getSessionUser() {
            isAuthenticated = true
            userEmail = user.email
        } else {
            isAuthenticated = false
        }
    }
    
    func signUp(email: String, password: String) {
        AuthService.shared.signUp(email: email, password: password) { [weak self] result in
            DispatchQueue.main.async {
                switch result {
                case .success(let user):
                    self?.isAuthenticated = true
                    self?.userEmail = user.email
                    print("User signed up: \(user.email)")
                case .failure(let error):
                    self?.errorMessage = error.localizedDescription
                }
            }
        }
    }
    
    func signIn(email: String, password: String) {
        AuthService.shared.signIn(email: email, password: password) { [weak self] result in
            DispatchQueue.main.async {
                switch result {
                case .success(let user):
                    self?.isAuthenticated = true
                    self?.userEmail = user.email
                    print("User signed in: \(user.email)")
                case .failure(let error):
                    self?.errorMessage = error.localizedDescription
                }
            }
        }
    }
    
    func signOut() {
        AuthService.shared.signOut { [weak self] result in
            DispatchQueue.main.async {
                switch result {
                case .success:
                    self?.isAuthenticated = false
                    self?.userEmail = nil
                    print("User signed out")
                case .failure(let error):
                    self?.errorMessage = error.localizedDescription
                }
            }
        }
    }
}
```

#### 9. **ViewModels/ConversationViewModel.swift**

```swift
import Foundation
import Combine

class ConversationViewModel: ObservableObject {
    private let realtimeService = RealtimeAPIService()
    private let audioService = AudioService()
    @Published var transcription: String = ""
    @Published var isRecording: Bool = false
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        setupBindings()
        realtimeService.connect()
    }
    
    func setupBindings() {
        audioService.onAudioDataAvailable = { [weak self] data in
            self?.realtimeService.sendAudioData(data)
        }
        
        realtimeService.onReceiveAudio = { [weak self] data in
            self?.audioService.playAudioData(data)
        }
        
        realtimeService.onReceiveTranscription = { [weak self] text in
            DispatchQueue.main.async {
                self?.transcription += text + "\n"
            }
        }
        
        realtimeService.onError = { error in
            print("Realtime API Error: \(error.localizedDescription)")
        }
    }
    
    func startOrStopRecording() {
        if isRecording {
            audioService.stopRecording()
        } else {
            audioService.startRecording()
        }
        isRecording.toggle()
    }
}
```

#### 10. **Views/LoginView.swift**

```swift
import SwiftUI

struct LoginView: View {
    @EnvironmentObject var authViewModel: AuthViewModel
    @State private var email = ""
    @State private var password = ""
    @State private var showSignUp = false
    
    var body: some View {
        NavigationView {
            VStack {
                Text("Welcome to MediTalk")
                    .font(.largeTitle)
                    .padding()
                
                Image(systemName: "stethoscope.circle.fill")
                    .resizable()
                    .frame(width: 100, height: 100)
                    .foregroundColor(.blue)
                    .padding()
                
                TextField("Email", text: $email)
                    .padding()
                    .background(Color(.secondarySystemBackground))
                    .cornerRadius(5.0)
                    .autocapitalization(.none)
                    .keyboardType(.emailAddress)
                
                SecureField("Password", text: $password)
                    .padding()
                    .background(Color(.secondarySystemBackground))
                    .cornerRadius(5.0)
                
                if let errorMessage = authViewModel.errorMessage {
                    Text(errorMessage)
                        .foregroundColor(.red)
                }
                
                Button(action: {
                    authViewModel.signIn(email: email, password: password)
                }) {
                    Text("Login")
                        .frame(maxWidth: .infinity)
                        .padding()
                        .background(Color.blue)
                        .foregroundColor(.white)
                        .cornerRadius(5.0)
                }
                .padding(.top)
                
                Button(action: {
                    showSignUp.toggle()
                }) {
                    Text("Don't have an account? Sign Up")
                        .foregroundColor(.blue)
                }
                .padding()
                .sheet(isPresented: $showSignUp) {
                    SignUpView()
                        .environmentObject(authViewModel)
                }
            }
            .padding()
            .navigationBarHidden(true)
        }
    }
}
```

#### 11. **Views/SignUpView.swift**

```swift
import SwiftUI

struct SignUpView: View {
    @EnvironmentObject var authViewModel: AuthViewModel
    @Environment(\.presentationMode) var presentationMode
    @State private var email = ""
    @State private var password = ""
    @State private var confirmPassword = ""
    
    var body: some View {
        VStack {
            Text("Create an Account")
                .font(.largeTitle)
                .padding()
            
            Image(systemName: "person.crop.circle.badge.plus")
                .resizable()
                .frame(width: 100, height: 100)
                .foregroundColor(.green)
                .padding()
            
            TextField("Email", text: $email)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(5.0)
                .autocapitalization(.none)
                .keyboardType(.emailAddress)
            
            SecureField("Password", text: $password)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(5.0)
            
            SecureField("Confirm Password", text: $confirmPassword)
                .padding()
                .background(Color(.secondarySystemBackground))
                .cornerRadius(5.0)
            
            if let errorMessage = authViewModel.errorMessage {
                Text(errorMessage)
                    .foregroundColor(.red)
            }
            
            Button(action: {
                guard password == confirmPassword else {
                    authViewModel.errorMessage = "Passwords do not match."
                    return
                }
                authViewModel.signUp(email: email, password: password)
            }) {
                Text("Sign Up")
                    .frame(maxWidth: .infinity)
                    .padding()
                    .background(Color.green)
                    .foregroundColor(.white)
                    .cornerRadius(5.0)
            }
            .padding(.top)
            .onReceive(authViewModel.$isAuthenticated) { isAuthenticated in
                if isAuthenticated {
                    self.presentationMode.wrappedValue.dismiss()
                }
            }
        }
        .padding()
    }
}
```

#### 12. **Views/MainView.swift**

```swift
import SwiftUI

struct MainView: View {
    @EnvironmentObject var authViewModel: AuthViewModel
    @StateObject private var conversationViewModel = ConversationViewModel()
    @State private var showSettings = false
    @State private var showConversation = false
    
    var body: some View {
        NavigationView {
            VStack {
                Text("MediTalk")
                    .font(.largeTitle)
                    .padding()
                
                Spacer()
                
                Button(action: {
                    conversationViewModel.startOrStopRecording()
                }) {
                    Image(systemName: conversationViewModel.isRecording ? "mic.circle.fill" : "mic.circle")
                        .resizable()
                        .frame(width: 150, height: 150)
                        .foregroundColor(conversationViewModel.isRecording ? .red : .blue)
                }
                
                Spacer()
                
                Button(action: {
                    showConversation.toggle()
                }) {
                    Text("View Conversation")
                }
                .padding()
                .sheet(isPresented: $showConversation) {
                    ConversationView()
                        .environmentObject(conversationViewModel)
                }
                
                HStack {
                    Button(action: {
                        showSettings.toggle()
                    }) {
                        Image(systemName: "gearshape")
                            .resizable()
                            .frame(width: 30, height: 30)
                    }
                    .sheet(isPresented: $showSettings) {
                        SettingsView()
                    }
                    
                    Spacer()
                    
                    Button(action: {
                        authViewModel.signOut()
                    }) {
                        Text("Logout")
                            .foregroundColor(.red)
                    }
                }
                .padding()
            }
            .navigationBarHidden(true)
            .onReceive(authViewModel.$isAuthenticated) { isAuthenticated in
                if !isAuthenticated {
                    // Handle user logout if necessary
                }
            }
        }
    }
}
```

#### 13. **Views/ConversationView.swift**

```swift
import SwiftUI

struct ConversationView: View {
    @EnvironmentObject var conversationViewModel: ConversationViewModel
    
    var body: some View {
        VStack {
            Text("Conversation")
                .font(.title)
                .padding()
            
            ScrollView {
                Text(conversationViewModel.transcription)
                    .padding()
            }
            
            Spacer()
            
            Button(action: {
                conversationViewModel.startOrStopRecording()
            }) {
                Image(systemName: conversationViewModel.isRecording ? "mic.circle.fill" : "mic.circle")
                    .resizable()
                    .frame(width: 100, height: 100)
                    .foregroundColor(conversationViewModel.isRecording ? .red : .blue)
            }
            .padding()
        }
    }
}
```

#### 14. **Views/SettingsView.swift**

```swift
import SwiftUI

struct SettingsView: View {
    @State private var selectedLanguage = "Spanish"
    let languages = ["Spanish", "Russian", "French", "German"]
    
    var body: some View {
        NavigationView {
            Form {
                Section(header: Text("Language Settings")) {
                    Picker("Target Language", selection: $selectedLanguage) {
                        ForEach(languages, id: \.self) {
                            Text($0)
                        }
                    }
                }
                
                Section(header: Text("Voice Settings")) {
                    Toggle("Enable Voice Feedback", isOn: .constant(true))
                    Slider(value: .constant(1.0), in: 0.5...2.0, step: 0.1) {
                        Text("Speech Rate")
                    }
                }
            }
            .navigationBarTitle("Settings", displayMode: .inline)
        }
    }
}
```

---

### Notes and Considerations

- **Placeholders:** Remember to replace `"YOUR_SUPABASE_API_KEY"`, `"YOUR_SUPABASE_URL"`, and `"YOUR_OPENAI_API_KEY"` with your actual Supabase and OpenAI credentials.
- **Dependencies:**
  - **Supabase SDK:** You'll need to install the Supabase Swift SDK. You can do this via Swift Package Manager by adding `https://github.com/supabase-community/supabase-swift` to your project.
  - **Starscream Library:** Install via Swift Package Manager by adding `https://github.com/daltoniam/Starscream`.
- **Audio Permissions:** Ensure your app's `Info.plist` includes the necessary permissions for microphone and audio usage, such as `NSMicrophoneUsageDescription`.
- **UI/UX Enhancements:**
  - Customize the UI elements to match your branding and design preferences.
  - Add animations and transitions for a smoother user experience.
- **Error Handling:** Implement comprehensive error handling and user feedback mechanisms for a robust application.
- **Testing:** Thoroughly test the application on actual devices to ensure functionality and performance meet expectations.
- **Future Enhancements:**
  - Implement encryption and advanced security measures.
  - Expand language support and add more customization options.
  - Integrate analytics and monitoring tools.
