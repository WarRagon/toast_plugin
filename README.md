# toast_plugin
A sample project demonstrating Android/iOS native Toast using Flutter and Svelte + Tauri. This repository showcases how to display Toast messages in Android using native code (Kotlin/Swift) within Flutter and Svelte + Tauri environments.

## flutter_application
flutter pub get

F5

### flutter native
android/app/src/main/kotlin/com/example/flutter_application/MainActivity.kt
```
package com.example.flutter_application

import android.widget.Toast
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity : FlutterActivity() {
    private val CHANNEL = "toast_channel"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, CHANNEL).setMethodCallHandler { call, result ->
            when (call.method) {
                "showToast" -> {
                    val message = call.argument<String>("message") ?: "message is null"
                    showToast(message)
                    result.success(null)
                }
                else -> result.notImplemented()
            }
        }
    }

    private fun showToast(message: String) {
        runOnUiThread {
            Toast.makeText(
                this@MainActivity,
                message,
                Toast.LENGTH_SHORT
            ).show()
        }
    }
}
```
ios/Runner/AppDelegate.swift
```
import Flutter
import UIKit

@main
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
        
        let controller: FlutterViewController = window?.rootViewController as! FlutterViewController
        let toastChannel = FlutterMethodChannel(name: "toast_channel",
                                                binaryMessenger: controller.binaryMessenger)
        
        toastChannel.setMethodCallHandler { [weak self] (call: FlutterMethodCall, result: @escaping FlutterResult) in
            switch call.method {
            case "showToast":
                if let args = call.arguments as? [String: Any],
                   let message = args["message"] as? String {
                    self?.showToast(message: message)
                    result(nil)
                } else {
                    result(FlutterError(code: "INVALID_ARGUMENTS",
                                        message: "Invalid arguments",
                                        details: nil))
                }
            default:
                result(FlutterMethodNotImplemented)
            }
        }
        
        GeneratedPluginRegistrant.register(with: self)
        return super.application(application, didFinishLaunchingWithOptions: launchOptions)
    }
    
    private func showToast(message: String) {
        DispatchQueue.main.async {
            if let rootVC = UIApplication.shared.windows.first(where: { $0.isKeyWindow })?.rootViewController {
                let alert = UIAlertController(title: nil, message: message, preferredStyle: .alert)
                rootVC.present(alert, animated: true)
                
                DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) {
                    alert.dismiss(animated: true)
                }
            }
        }
    }
}
```
## tauri-app
tauri-app/tauri-plugin-example
```
pnpm install

pnpm build
```
tauri-app
```
pnpm install

pnpm tauri android init

pnpm tauri android dev

pnpm tauri ios init

pnpm tauri ios dev
```

### tauri-app plugin example
tauri-app
```
pnpm tauri plugin new --ios --android example
```
tauri-app/tauri-plugin-example
```
pnpm install

pnpm run build
```
tauri-app/src-tauri/Carog.toml
```
[dependencies]
tauri-plugin-example = { path = "../tauri-plugin-example/" }
```
tauri-app/package.json
```
"dependencies": {
    "tauri-plugin-example-api": "file:./tauri-plugin-example"
}
```
tauri-app/src-tauri/src/lib.rs
```
tauri::Builder::default()
    .plugin(tauri_plugin_example::init())
```
tauri-app/src-tauri/capabilities/default.json
```
"permissions": [
    "example:default"
]
```
tauri-app
```
pnpm install

pnpm tauri android dev

pnpm tauri ios dev
```
tauri-app/tauri-plugin-example/android/src/main/java/ExamplePlugin.kt
```
import android.widget.Toast

@InvokeArg
class ToastArgs {
  var value: String? = null
}

@TauriPlugin
class ExamplePlugin(private val activity: Activity) : Plugin(activity) {
  @Command
  fun toast(invoke: Invoke) {
    val args = invoke.parseArgs(PingArgs::class.java)

    Toast.makeText(activity, args.value, Toast.LENGTH_SHORT).show()
  }
}
```
tauri-app/tauri-plugin-example/ios/Sources/ExamplePlugin.swift
```
class ToastArgs: Decodable {
    let value: String?
}

class ExamplePlugin: Plugin {
  @objc public func toast(_ invoke: Invoke) throws {
    let args = try invoke.parseArgs(ToastArgs.self)
    let message = args.value ?? "Default message"
    
    DispatchQueue.main.async {
      let alert = UIAlertController(
        title: nil, 
        message: message, 
        preferredStyle: .alert
      )
      
      if let rootVC = UIApplication.shared.windows.first(where: { $0.isKeyWindow })?.rootViewController {
        rootVC.present(alert, animated: true)
        
        DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) {
          alert.dismiss(animated: true)
        }
      }
      
      invoke.resolve()
    }
  }
}
```
tauri-app/tauri-plugin-example/src/models.rs
```
#[derive(Debug, Deserialize, Serialize)]
#[serde(rename_all = "camelCase")]
pub struct ToastRequest {
  pub value: String,
}
```
tauri-app/tauri-plugin-example/src/commands.rs
```
#[command]
pub(crate) async fn toast<R: Runtime>(
    app: AppHandle<R>,
    payload: ToastRequest,
) -> Result<()> {
    app.example().toast(payload)
}
```
tauri-app/tauri-plugin-example/src/mobile.rs
```
impl<R: Runtime> Example<R> {
  pub fn toast(&self, payload: ToastRequest) -> crate::Result<()> {
    self.0
        .run_mobile_plugin("toast", payload)
        .map_err(Into::into)
  }
}
```
tauri-app/tauri-plugin-example/guest-js/index.ts
```
export async function toast(value: string): Promise<void> {
  await invoke<{ value?: string }>("plugin:example|toast", {
      payload: {
          value,
      },
  }).then((r) => (r.value ? r.value : null));
}
```
tauri-app/tauri-plugin-example/build.rs
```
const COMMANDS: &[&str] = &["ping"];
->
const COMMANDS: &[&str] = &["ping", "toast"];
```
tauri-app/tauri-plugin-example/permissions/default.toml
```
permissions = ["allow-ping", "allow-toast"]
->
permissions = ["allow-ping", "allow-toast"]
```
tauri-app/tauri-plugin-example
```
pnpm build
```
tauri-app
```
pnpm remove tauri-plugin-example-api
```
tauri-app/package.json
```
"dependencies": {
    "tauri-plugin-example-api": "file:./tauri-plugin-example"
}
```
tauri-app
```
pnpm install
```