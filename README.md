import tkinter as tk
from tkinter import scrolledtext, ttk, filedialog, messagebox
import requests
import threading
import time
import re

class RedditPro:
    def __init__(self, root):
        self.root = root
        self.root.title("Reddit Tracker")
        self.root.geometry("360x600")
        self.root.attributes('-topmost', True) 
        self.root.configure(bg="#1A1A1B")

        # 1. Action Bar
        btn_frame = tk.Frame(root, bg="#1A1A1B")
        btn_frame.pack(fill=tk.X, pady=5)
        
        self.btn_run = tk.Button(btn_frame, text="▶ START", command=self.start, bg="#FF4500", fg="white", font=("Verdana", 8, "bold"), bd=0, height=2, cursor="hand2")
        self.btn_run.pack(side=tk.LEFT, fill=tk.X, expand=True, padx=5)
        
        # Compact Utility Buttons
        tk.Button(btn_frame, text="Clear", command=lambda: self.input.delete("1.0", tk.END), font=("Verdana", 8), bg="#333F44", fg="white", bd=0).pack(side=tk.LEFT, padx=2)
        tk.Button(btn_frame, text="Save", command=self.save_file, font=("Verdana", 8), bg="#333F44", fg="white", bd=0).pack(side=tk.LEFT, padx=2)
        tk.Button(btn_frame, text="Copy", command=self.copy, font=("Verdana", 8), bg="#333F44", fg="white", bd=0).pack(side=tk.LEFT, padx=(2, 5))

        # 2. Bulk Input
        tk.Label(root, text="SUBREDDIT LIST:", font=("Verdana", 7, "bold"), bg="#1A1A1B", fg="#818384").pack(anchor="w", padx=10)
        self.input = tk.Text(root, height=5, font=("Consolas", 10), bg="#272729", fg="#D7DADC", insertbackground="white", bd=0, pady=5)
        self.input.pack(fill=tk.X, padx=10, pady=5)

        # 3. Buffer Countdown & Progress
        self.timer_label = tk.Label(root, text="Ready", font=("Verdana", 8, "italic"), bg="#1A1A1B", fg="#FF4500")
        self.timer_label.pack(pady=2)
        
        style = ttk.Style()
        style.theme_use('default')
        style.configure("Reddit.Horizontal.TProgressbar", thickness=4, troughcolor='#272729', background='#FF4500', bordercolor="#1A1A1B")
        self.prog = ttk.Progressbar(root, orient="horizontal", mode="determinate", style="Reddit.Horizontal.TProgressbar")
        self.prog.pack(fill=tk.X, padx=10, pady=2)

        # 4. Results Area
        self.output = scrolledtext.ScrolledText(root, font=("Segoe UI", 9), bg="#1A1A1B", fg="#D7DADC", bd=0, highlightthickness=0)
        self.output.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        self.output.tag_config("sub", font=("Segoe UI", 10, "bold"), foreground="#FF4500")
        self.output.tag_config("ups", foreground="#D7DADC", font=("Segoe UI", 8, "bold"))
        self.output.tag_config("title", foreground="#D7DADC")

    def save_file(self):
        """Opens a dialog to save the output to a specific location."""
        content = self.output.get(1.0, tk.END).strip()
        if not content:
            messagebox.showwarning("Empty", "Nothing to save yet!")
            return
        
        file_path = filedialog.asksaveasfilename(
            defaultextension=".txt",
            filetypes=[("Text files", "*.txt"), ("All files", "*.*")],
            title="Save Reddit Report"
        )
        if file_path:
            with open(file_path, "w", encoding="utf-8") as f:
                f.write(content)
            self.timer_label.config(text="File Saved Successfully!")

    def copy(self):
        self.root.clipboard_clear()
        self.root.clipboard_append(self.output.get(1.0, tk.END))

    def clean_name(self, text):
        text = text.strip().lower()
        if 'reddit.com/r/' in text: text = text.split('reddit.com/r/')[-1]
        elif 'r/' in text: text = text.split('r/')[-1]
        return text.split('/')[0].split('?')[0].strip()

    def start(self):
        raw_input = self.input.get(1.0, tk.END)
        subs = [self.clean_name(line) for line in raw_input.split('\n') if line.strip()]
        subs = [s for s in list(dict.fromkeys(subs)) if s and s != "reddit.com"]
        
        if not subs: return
        self.btn_run.config(state="disabled", bg="#444")
        self.output.delete(1.0, tk.END)
        threading.Thread(target=self.fetch, args=(subs,), daemon=True).start()

    def fetch(self, subs):
        headers = {"User-Agent": "RedditMiniPro/3.1"}
        for i, sub in enumerate(subs):
            self.timer_label.config(text=f"Fetching r/{sub}...")
            try:
                r = requests.get(f"https://www.reddit.com/r/{sub}/top.json?t=week&limit=5", headers=headers, timeout=10)
                if r.status_code == 200:
                    self.output.insert(tk.END, f"r/{sub.upper()}\n", "sub")
                    posts = r.json()['data']['children']
                    for p in posts:
                        data = p['data']
                        ups = f"{data['ups']:,}"
                        self.output.insert(tk.END, f" ▲ {ups}", "ups")
                        self.output.insert(tk.END, f" | {data['title']}\n", "title")
                    self.output.insert(tk.END, "\n")
                else:
                    self.output.insert(tk.END, f" × r/{sub} unavailable\n\n")
            except:
                self.output.insert(tk.END, f" ! Error connecting to r/{sub}\n\n")
            
            self.prog['value'] = ((i + 1) / len(subs)) * 100
            self.output.see(tk.END)

            if i < len(subs) - 1:
                for remaining in range(10, 0, -1):
                    self.timer_label.config(text=f"Next fetch in {remaining}s...")
                    time.sleep(1)
        
        self.timer_label.config(text="All Tasks Complete")
        self.btn_run.config(state="normal", bg="#FF4500")

if __name__ == "__main__":
    root = tk.Tk()
    app = RedditPro(root)
    root.mainloop()
