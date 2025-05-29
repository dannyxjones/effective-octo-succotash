seriously just wanting to share this simple program that turns off touchscreen. everybody has trust issues now.
Run as admin 
push the button to turn on
push the button again to turn off

below is the code I compiled to a .exe
import tkinter as tk
import subprocess
import threading
import ctypes

class TouchscreenToggle:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("Legion Go Touchscreen Toggle")
        self.root.geometry("150x80")
        self.root.resizable(False, False)
        self.root.attributes('-topmost', True)
        
        self.touchscreen_enabled = True
        self.device_id = None
        self.setup_ui()
        
    def setup_ui(self):
        self.toggle_button = tk.Button(
            self.root,
            text="Push Me",
            font=("Arial", 12, "bold"),
            bg="purple",
            fg="white",
            command=self.toggle_touchscreen,
            relief="raised",
            bd=3
        )
        self.toggle_button.pack(fill=tk.BOTH, expand=True, padx=2, pady=2)
    
    def is_admin(self):
        try:
            return ctypes.windll.shell32.IsUserAnAdmin()
        except:
            return False
    
    def find_touchscreen_device(self):
        try:
            # Method 1: Comprehensive search
            ps_script = '''
            Get-PnpDevice | Where-Object {
                ($_.FriendlyName -like "*touch*" -or 
                 $_.FriendlyName -like "*HID*" -or
                 $_.Class -eq "HIDClass") -and
                $_.Status -eq "OK"
            } | Select-Object InstanceId, FriendlyName, Class | Format-List
            '''
            
            result = subprocess.run(
                ['powershell', '-Command', ps_script],
                capture_output=True,
                text=True,
                timeout=10
            )
            
            if result.returncode == 0:
                lines = result.stdout.split('\n')
                for i, line in enumerate(lines):
                    if "InstanceId" in line and "HID\\" in line:
                        instance_id = line.split(':')[-1].strip()
                        # Check nearby lines for touchscreen confirmation
                        for j in range(i+1, min(len(lines), i+5)):
                            if "FriendlyName" in lines[j] and ("touch" in lines[j].lower() or "input" in lines[j].lower()):
                                return instance_id
            
            # Method 2: Direct HID search
            ps_script2 = '''
            Get-PnpDevice -Class "HIDClass" | Where-Object {
                $_.InstanceId -like "*VID_3938*" -or
                $_.InstanceId -like "*touch*" -or
                ($_.FriendlyName -like "*HID-compliant touch*")
            } | Select-Object -First 1 -ExpandProperty InstanceId
            '''
            
            result = subprocess.run(
                ['powershell', '-Command', ps_script2],
                capture_output=True,
                text=True,
                timeout=10
            )
            
            if result.returncode == 0 and result.stdout.strip():
                return result.stdout.strip()
                
        except Exception as e:
            print(f"Error finding device: {e}")
        
        # Fallback device ID
        return "HID\\VID_3938&PID_1311&MI_01&COL04&430"
    
    def toggle_touchscreen_method1(self, device_id):
        """PowerShell Disable/Enable-PnpDevice method"""
        try:
            cmd = ('Disable' if self.touchscreen_enabled else 'Enable') + f'-PnpDevice -InstanceId "{device_id}" -Confirm:$false'
            result = subprocess.run(
                ['powershell', '-Command', cmd],
                capture_output=True,
                text=True,
                timeout=15
            )
            return result.returncode == 0, result.stderr
        except Exception as e:
            return False, str(e)
    
    def toggle_touchscreen_method2(self, device_id):
        """PnpUtil command line method"""
        try:
            cmd = f'pnputil /{"disable" if self.touchscreen_enabled else "enable"}-device "{device_id}"'
            result = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=15)
            return result.returncode == 0, result.stderr
        except Exception as e:
            return False, str(e)
    
    def toggle_touchscreen_method3(self, device_id):
        """Registry modification method"""
        try:
            if self.touchscreen_enabled:
                cmd = f'''
                $deviceId = "{device_id}"
                $regPath = "HKLM:\\SYSTEM\\CurrentControlSet\\Enum\\$deviceId"
                if (Test-Path $regPath) {{
                    Set-ItemProperty -Path "$regPath" -Name "ConfigFlags" -Value 0x20 -Type DWord -Force
                }}'''
            else:
                cmd = f'''
                $deviceId = "{device_id}"
                $regPath = "HKLM:\\SYSTEM\\CurrentControlSet\\Enum\\$deviceId"
                if (Test-Path $regPath) {{
                    Remove-ItemProperty -Path "$regPath" -Name "ConfigFlags" -ErrorAction SilentlyContinue
                }}'''
            
            result = subprocess.run(
                ['powershell', '-Command', cmd],
                capture_output=True,
                text=True,
                timeout=15
            )
            
            if result.returncode == 0:
                subprocess.run(
                    ['powershell', '-Command', 'pnputil /scan-devices'],
                    capture_output=True,
                    timeout=10
                )
            
            return result.returncode == 0, result.stderr
        except Exception as e:
            return False, str(e)
    
    def toggle_touchscreen(self):
        def worker():
            if not self.is_admin():
                print("Error: Admin privileges required")
                return
            
            if not self.device_id:
                self.device_id = self.find_touchscreen_device()
                print(f"Found device ID: {self.device_id}")
            
            if not self.device_id:
                print("Error: No touchscreen device found")
                return
            
            methods = [
                ("PowerShell PnpDevice", self.toggle_touchscreen_method1),
                ("PnpUtil", self.toggle_touchscreen_method2),
                ("Registry", self.toggle_touchscreen_method3)
            ]
            
            for name, method in methods:
                success, error = method(self.device_id)
                if success:
                    print(f"Success using {name} method")
                    self.touchscreen_enabled = not self.touchscreen_enabled
                    self.root.after(0, self.update_ui)
                    break
                else:
                    print(f"Failed with {name}: {error}")
        
        # Update UI immediately
        self.toggle_button.config(bg="gray", text="...")
        
        # Run in thread
        threading.Thread(target=worker, daemon=True).start()
        
        # Reset button text after timeout
        self.root.after(2000, lambda: self.toggle_button.config(text="Push Me"))
    
    def update_ui(self):
        self.toggle_button.config(
            bg="purple" if self.touchscreen_enabled else "red",
            text="Push Me"
        )
    
    def run(self):
        self.root.mainloop()

if __name__ == "__main__":
    app = TouchscreenToggle()
    app.run()
