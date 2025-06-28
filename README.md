# File-organizer
Smart File Organizer. A Python app with a modern Tkinter GUI that sorts files in a selected folder by extension. It includes real-time logs, an undo function, and supports stopping the process at any time.

import os
import shutil
from datetime import datetime
import tkinter as tk
from tkinter import filedialog, messagebox, scrolledtext
import threading

# === FILE ORGANIZER CLASS (nemodificat) ===
class FileOrganizer:
    def __init__(self, folder_path, log_widget=None):
        self.folder_path = folder_path
        self.files = []
        self.log_file = os.path.join(folder_path, 'moved_files.log')
        self.log_widget = log_widget

    def log(self, message):
        if self.log_widget:
            self.log_widget.insert(tk.END, message + '\n')
            self.log_widget.see(tk.END)
        print(message)

    def scan_files(self):
        if not os.path.exists(self.folder_path):
            self.log(f"❌ Folderul nu există: {self.folder_path}")
            return False
        if not os.path.isdir(self.folder_path):
            self.log(f"❌ Calea nu este un folder: {self.folder_path}")
            return False

        self.files = [
            f for f in os.listdir(self.folder_path)
            if os.path.isfile(os.path.join(self.folder_path, f))
        ]
        self.log(f"📂 Fișiere găsite: {len(self.files)}")
        for f in self.files:
            self.log(" - " + f)
        return True

    def log_move(self, source, destination):
        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        with open(self.log_file, 'a') as f:
            f.write(f"{timestamp} | {source} -> {destination}\n")

    def organize_files(self, stop_flag=None):
        for file_name in self.files:
            if stop_flag and stop_flag.is_set():
                self.log("🛑 Organizare întreruptă de utilizator.")
                break

            _, ext = os.path.splitext(file_name)
            ext = ext[1:].lower() or "no_extension"

            source_path = os.path.join(self.folder_path, file_name)
            dest_folder = os.path.join(self.folder_path, ext)
            os.makedirs(dest_folder, exist_ok=True)

            dest_path = os.path.join(dest_folder, file_name)
            name, ext_dot = os.path.splitext(file_name)
            counter = 1
            while os.path.exists(dest_path):
                dest_path = os.path.join(dest_folder, f"{name}_{counter}{ext_dot}")
                counter += 1

            try:
                shutil.move(source_path, dest_path)
                self.log(f"✅ Mutat: {file_name} → {ext}/")
                self.log_move(source_path, dest_path)
            except PermissionError:
                self.log(f"🚫 Permisiune refuzată: {file_name}")
            except Exception as e:
                self.log(f"❌ Eroare la mutarea {file_name}: {e}")

    def undo_last_moves(self):
        if not os.path.exists(self.log_file):
            self.log("❌ Nu există fișier de log.")
            return

        with open(self.log_file, 'r') as f:
            lines = f.readlines()

        for line in reversed(lines):
            try:
                _, move_info = line.split('|')
                src, dest = move_info.strip().split('->')
                if os.path.exists(dest.strip()):
                    shutil.move(dest.strip(), src.strip())
                    self.log(f"↩️ Undo: {os.path.basename(src.strip())}")
            except:
                self.log(f"⚠️ Eroare la undo: {line.strip()}")

        with open(self.log_file, 'w') as f:
            f.truncate(0)
        self.log("✅ Undo complet.")

# === APP CLASS CU DESIGN MODERN ===
class App:
    def __init__(self, root):
        self.root = root
        self.root.title("🧠 Smart File Organizer")
        self.root.geometry("700x550")
        self.root.configure(bg="#f2f2f2")
        self.root.resizable(False, False)

        self.folder_path = None
        self.organizer = None
        self.stop_flag = threading.Event()

        font_title = ("Segoe UI", 14, "bold")
        font_normal = ("Segoe UI", 11)

        # === INTERFAȚĂ ===
        tk.Label(root, text="Smart File Organizer", font=font_title, bg="#f2f2f2").pack(pady=10)

        self.btn_select = tk.Button(root, text="📁 Selectează folder", font=font_normal, bg="#4285f4", fg="white", command=self.select_folder)
        self.btn_select.pack(pady=8)

        self.lbl_folder = tk.Label(root, text="Niciun folder selectat", font=("Segoe UI", 10), bg="#f2f2f2", fg="gray")
        self.lbl_folder.pack()

        self.btn_start = tk.Button(root, text="▶️ Organizează fișierele", font=font_normal, command=self.start_organize, bg="#34a853", fg="white", state=tk.DISABLED)
        self.btn_start.pack(pady=5)

        self.btn_stop = tk.Button(root, text="⛔ Stop", font=font_normal, command=self.stop_organize, bg="#ea4335", fg="white", state=tk.DISABLED)
        self.btn_stop.pack(pady=5)

        self.btn_undo = tk.Button(root, text="↩️ Undo ultima organizare", font=font_normal, command=self.undo, bg="#fbbc05", fg="black", state=tk.DISABLED)
        self.btn_undo.pack(pady=5)

        self.log_text = scrolledtext.ScrolledText(root, width=80, height=18, font=("Consolas", 10), bg="white", fg="black")
        self.log_text.pack(padx=10, pady=10)

    def select_folder(self):
        folder = filedialog.askdirectory()
        if folder:
            self.folder_path = folder
            self.lbl_folder.config(text=folder, fg="black")
            self.organizer = FileOrganizer(folder, log_widget=self.log_text)
            self.btn_start.config(state=tk.NORMAL)
            self.btn_undo.config(state=tk.NORMAL)
            self.log(f"📂 Folder selectat: {folder}")

    def start_organize(self):
        if not self.organizer:
            messagebox.showerror("Eroare", "Selectează un folder.")
            return

        self.stop_flag.clear()
        self.btn_start.config(state=tk.DISABLED)
        self.btn_stop.config(state=tk.NORMAL)
        self.log("▶️ Încep organizarea...")

        threading.Thread(target=self._run_organize, daemon=True).start()

    def _run_organize(self):
        if self.organizer.scan_files():
            self.organizer.organize_files(stop_flag=self.stop_flag)
        self.log("✅ Organizare finalizată.\n")
        self.btn_start.config(state=tk.NORMAL)
        self.btn_stop.config(state=tk.DISABLED)

    def stop_organize(self):
        self.log("⛔ Organizarea a fost oprită de utilizator.")
        self.stop_flag.set()
        self.btn_stop.config(state=tk.DISABLED)

    def undo(self):
        if not self.organizer:
            messagebox.showerror("Eroare", "Selectează un folder.")
            return
        self.log("↩️ Încep undo...")
        self.organizer.undo_last_moves()
        self.log("✅ Undo complet.\n")

    def log(self, message):
        self.log_text.insert(tk.END, message + "\n")
        self.log_text.see(tk.END)

if __name__ == "__main__":
    root = tk.Tk()
    app = App(root)
    root.mainloop()



<img width="608" alt="4" src="https://github.com/user-attachments/assets/80f1c6af-fc21-4870-8903-ccd06b6098f1" />
<img width="533" alt="3" src="https://github.com/user-attachments/assets/465c3fd4-b15a-4f87-bb4e-3c868f67860a" />
<img width="554" alt="2" src="https://github.com/user-attachments/assets/9f80d613-42ee-4a30-b25b-9de147d6f9ac" />
<img width="608" alt="1" src="https://github.com/user-attachments/assets/65913dd6-76bb-498d-84ab-397028aaa76e" />


