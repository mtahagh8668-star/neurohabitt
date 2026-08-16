"""
NeuroHabit - نسخه نهایی ۵.۰ با Smart Break Reminder و AppMonitor
برنامه کامل مدیریت بهره‌وری با تایمر استراحت هوشمند
"""

import sys
import os
import threading
import time
import tkinter as tk
from tkinter import ttk, messagebox, filedialog, simpledialog
from datetime import datetime
import json
import winsound

# ===========================
# تنظیمات مسیر
# ===========================

project_path = r"C:\neurohabit"
if project_path not in sys.path:
    sys.path.insert(0, project_path)

try:
    from modules.behavior.keyboard_monitor import KeyboardMonitor
    from modules.behavior.mouse_monitor import MouseMonitor
    from modules.behavior.app_monitor import AppMonitor
    from modules.data.logger import DataLogger
except ImportError as e:
    print(f"⚠️ خطا: {e}")
    print("لطفاً مطمئن شوید فایل‌های زیر وجود دارند:")
    print("  - modules/behavior/keyboard_monitor.py")
    print("  - modules/behavior/mouse_monitor.py")
    print("  - modules/behavior/app_monitor.py")
    print("  - modules/data/logger.py")
    sys.exit(1)

# ===========================
# تنظیمات گرافیکی
# ===========================

class Theme:
    BG = '#08080f'
    HEADER = '#0f0f23'
    CARD = '#151530'
    BORDER = '#252550'
    
    PRIMARY = '#6c5ce7'
    SUCCESS = '#00b894'
    DANGER = '#e17055'
    WARNING = '#fdcb6e'
    INFO = '#00cec9'
    GOLD = '#fdcb6e'
    PINK = '#fd79a8'
    CYAN = '#00cec9'
    
    TEXT = '#dfe6e9'
    TEXT_LIGHT = '#b2bec3'
    
    FONT_TITLE = ('Segoe UI', 18, 'bold')
    FONT_HEADING = ('Segoe UI', 11, 'bold')
    FONT_BODY = ('Segoe UI', 9)
    FONT_SMALL = ('Segoe UI', 8)
    FONT_LARGE = ('Segoe UI', 16, 'bold')
    FONT_MONO = ('Consolas', 9)
    
    BUTTON_STYLE = {
        'font': ('Segoe UI', 9, 'bold'),
        'relief': 'flat',
        'bd': 0,
        'cursor': 'hand2',
        'padx': 14,
        'pady': 5
    }

# ===========================
# کلاس Smart Reminder
# ===========================

class SmartReminder:
    def __init__(self, parent):
        self.parent = parent
        self.root = parent.root
        
        self.break_interval = 40
        self.sound_file = None
        self.is_running = False
        self.is_paused = False
        self.remaining_time = 0
        self.timer_thread = None
        self.stop_event = threading.Event()  # برای توقف سریع
        
        self._load_settings()
        
    def _load_settings(self):
        try:
            with open("data/reminder_settings.json", "r", encoding="utf-8") as f:
                settings = json.load(f)
                self.break_interval = settings.get("interval", 40)
                self.sound_file = settings.get("sound_file", None)
        except:
            pass
            
    def _save_settings(self):
        try:
            os.makedirs("data", exist_ok=True)
            with open("data/reminder_settings.json", "w", encoding="utf-8") as f:
                json.dump({
                    "interval": self.break_interval,
                    "sound_file": self.sound_file
                }, f, indent=2)
        except:
            pass
            
    def set_interval(self, minutes):
        self.break_interval = minutes
        self._save_settings()
        
    def set_sound(self, file_path):
        if file_path and os.path.exists(file_path):
            self.sound_file = file_path
            self._save_settings()
            return True
        return False
        
    def start(self):
        if self.is_running:
            return
            
        self.is_running = True
        self.is_paused = False
        self.remaining_time = self.break_interval * 60
        self.stop_event.clear()
        
        self.timer_thread = threading.Thread(target=self._timer_loop, daemon=True)
        self.timer_thread.start()
        
        # ===== به‌روزرسانی در Main Thread =====
        self.root.after(0, lambda: self.parent._add_log(
            f"⏱️ تایمر استراحت شروع شد ({self.break_interval} دقیقه)", "info"
        ))
        
    def pause(self):
        if self.is_running and not self.is_paused:
            self.is_paused = True
            self.root.after(0, lambda: self.parent._add_log("⏸️ تایمر استراحت متوقف شد", "warning"))
            
    def resume(self):
        if self.is_running and self.is_paused:
            self.is_paused = False
            self.root.after(0, lambda: self.parent._add_log("▶️ تایمر استراحت ادامه یافت", "info"))
            
    def stop(self):
        self.is_running = False
        self.is_paused = False
        self.remaining_time = 0
        self.stop_event.set()
        self.root.after(0, lambda: self.parent._add_log("⏹️ تایمر استراحت متوقف شد", "monitor"))
        
    def _timer_loop(self):
        """حلقه تایمر در Thread جداگانه - فقط داده رو تغییر میده"""
        while self.is_running and self.remaining_time > 0:
            if not self.is_paused:
                # ===== فقط یک ثانیه صبر کن =====
                if self.stop_event.wait(1):
                    break
                self.remaining_time -= 1
                
                # ===== به‌روزرسانی UI در Main Thread =====
                self.root.after(0, self._update_display)
                
        # ===== وقتی تایمر تموم شد =====
        if self.is_running and self.remaining_time <= 0:
            self.root.after(0, self._on_time_up)
            
    def _update_display(self):
        """به‌روزرسانی نمایش (در Main Thread اجرا میشه)"""
        if hasattr(self.parent, 'reminder_label'):
            mins = self.remaining_time // 60
            secs = self.remaining_time % 60
            self.parent.reminder_label.config(text=f"⏱️ {mins:02d}:{secs:02d}")
            
    def _on_time_up(self):
        """پایان تایمر (در Main Thread اجرا میشه)"""
        self.is_running = False
        self._play_sound()
        
        response = messagebox.askyesno(
            "⏰ زمان استراحت!",
            f"{self.break_interval} دقیقه کار کردی!\n\n"
            "وقتشه یه استراحت کوتاه داشته باشی.\n"
            "چشمانت رو ببند، بلند شو و یه قدم بزن.\n\n"
            "آیا می‌خواهی ادامه بدی؟",
            icon='question'
        )
        
        if response:
            self.parent._add_log("✅ کاربر ادامه داد", "success")
            self.start()
        else:
            exit_response = messagebox.askyesno(
                "🚪 خروج",
                "آیا قصد خروج از برنامه را داری؟",
                icon='question'
            )
            if exit_response:
                self.parent._on_closing()
            else:
                self.parent._add_log("⏸️ کاربر تصمیم گرفت بمونه", "info")
                self.start()
                
    def _play_sound(self):
        try:
            try:
                import pygame
                if self.sound_file and os.path.exists(self.sound_file):
                    pygame.mixer.init()
                    pygame.mixer.music.load(self.sound_file)
                    pygame.mixer.music.play()
                    return
            except:
                pass
                
            for _ in range(3):
                winsound.Beep(1000, 500)
                time.sleep(0.2)
            winsound.Beep(1500, 800)
        except:
            pass
        
    def _load_settings(self):
        try:
            with open("data/reminder_settings.json", "r", encoding="utf-8") as f:
                settings = json.load(f)
                self.break_interval = settings.get("interval", 40)
                self.sound_file = settings.get("sound_file", None)
        except:
            pass
            
    def _save_settings(self):
        try:
            os.makedirs("data", exist_ok=True)
            with open("data/reminder_settings.json", "w", encoding="utf-8") as f:
                json.dump({
                    "interval": self.break_interval,
                    "sound_file": self.sound_file
                }, f, indent=2)
        except:
            pass
            
    def set_interval(self, minutes):
        self.break_interval = minutes
        self._save_settings()
        
    def set_sound(self, file_path):
        if file_path and os.path.exists(file_path):
            self.sound_file = file_path
            self._save_settings()
            return True
        return False
        
    def start(self):
        if self.is_running:
            return
            
        self.is_running = True
        self.is_paused = False
        self.remaining_time = self.break_interval * 60
        
        self.timer_thread = threading.Thread(target=self._timer_loop, daemon=True)
        self.timer_thread.start()
        
        self.parent._add_log(f"⏱️ تایمر استراحت شروع شد ({self.break_interval} دقیقه)", "info")
        
    def pause(self):
        if self.is_running and not self.is_paused:
            self.is_paused = True
            self.parent._add_log("⏸️ تایمر استراحت متوقف شد", "warning")
            
    def resume(self):
        if self.is_running and self.is_paused:
            self.is_paused = False
            self.parent._add_log("▶️ تایمر استراحت ادامه یافت", "info")
            
    def stop(self):
        self.is_running = False
        self.is_paused = False
        self.remaining_time = 0
        self.parent._add_log("⏹️ تایمر استراحت متوقف شد", "monitor")
        
    def _timer_loop(self):
        while self.is_running and self.remaining_time > 0:
            if not self.is_paused:
                time.sleep(1)
                self.remaining_time -= 1
                self.root.after(0, self._update_display)
                
        if self.is_running and self.remaining_time <= 0:
            self.root.after(0, self._on_time_up)
            
    def _update_display(self):
        if hasattr(self.parent, 'reminder_label'):
            mins = self.remaining_time // 60
            secs = self.remaining_time % 60
            self.parent.reminder_label.config(text=f"⏱️ {mins:02d}:{secs:02d}")
            
    def _on_time_up(self):
        self.is_running = False
        self._play_sound()
        
        response = messagebox.askyesno(
            "⏰ زمان استراحت!",
            f"{self.break_interval} دقیقه کار کردی!\n\n"
            "وقتشه یه استراحت کوتاه داشته باشی.\n"
            "چشمانت رو ببند، بلند شو و یه قدم بزن.\n\n"
            "آیا می‌خواهی ادامه بدی؟",
            icon='question'
        )
        
        if response:
            self.parent._add_log("✅ کاربر ادامه داد", "success")
            self.start()
        else:
            exit_response = messagebox.askyesno(
                "🚪 خروج",
                "آیا قصد خروج از برنامه را داری؟",
                icon='question'
            )
            if exit_response:
                self.parent._on_closing()
            else:
                self.parent._add_log("⏸️ کاربر تصمیم گرفت بمونه", "info")
                self.start()
                
    def _play_sound(self):
        try:
            try:
                import pygame
                if self.sound_file and os.path.exists(self.sound_file):
                    pygame.mixer.init()
                    pygame.mixer.music.load(self.sound_file)
                    pygame.mixer.music.play()
                    return
            except:
                pass
                
            for _ in range(3):
                winsound.Beep(1000, 500)
                time.sleep(0.2)
            winsound.Beep(1500, 800)
        except:
            pass

# ===========================
# کلاس اصلی برنامه
# ===========================

class NeuroHabitApp:
    def __init__(self):
        # ===== پنجره اصلی =====
        self.root = tk.Tk()
        self.root.title("🧠 NeuroHabit Pro 5.0")
        self.root.geometry("1200x950")
        self.root.configure(bg=Theme.BG)
        self.root.minsize(1050, 850)
        
        # ===== متغیرهای اصلی =====
        self.auto_save_interval = 60
        self.is_auto_save_running = False
        self.is_monitoring = False
        self.session_start = None
        self.current_session_data = None
        self.history_data = []
        
        # ===== ماژول‌ها =====
        self.keyboard = KeyboardMonitor()
        self.mouse = MouseMonitor()
        self.logger = DataLogger()
        self.app_monitor = AppMonitor(parent=self)
        
        # ===== آمار لحظه‌ای =====
        self.live_cards = {}
        
        # ===== Activity Log =====
        self.activity_log = []
        self.max_log_entries = 50
        self.log_listbox = None
        
        # ===== Smart Reminder =====
        self.reminder = None
        
        # ===== بارگذاری تاریخچه =====
        self._load_history()
        
        # ===== راه‌اندازی UI =====
        self._setup_ui()
        self._setup_hotkeys()
        self._setup_protocols()
        
        # ===== شروع تایمرها =====
        self._update_timer()
        
        # ===== نمایش اولیه =====
        self._show_history_list()
        if self.history_data:
            self._show_session_info(self.history_data[0])
        
        self._add_log("🚀 NeuroHabit راه‌اندازی شد", "info")
        self._add_log("⏸️ آماده شروع", "info")
        
    # ===========================
    # بارگذاری تاریخچه
    # ===========================
    
    def _load_history(self):
        self.history_data = []
        summary_dir = "data/summary"
        if os.path.exists(summary_dir):
            try:
                files = sorted([f for f in os.listdir(summary_dir) if f.startswith('combined_')])
                for file in files[-20:]:
                    try:
                        with open(os.path.join(summary_dir, file), 'r', encoding='utf-8') as f:
                            data = json.load(f)
                            data['file'] = file
                            self.history_data.append(data)
                    except:
                        pass
            except:
                pass
        self.history_data.reverse()
    
    # ===========================
    # Activity Log
    # ===========================
    
    def _add_log(self, message, log_type="info"):
        timestamp = datetime.now().strftime("%H:%M:%S")
        colors = {
            'info': Theme.TEXT_LIGHT,
            'success': Theme.SUCCESS,
            'warning': Theme.WARNING,
            'error': Theme.DANGER,
            'monitor': Theme.PRIMARY,
            'save': Theme.GOLD,
            'system': Theme.INFO,
            'app': Theme.CYAN
        }
        color = colors.get(log_type, Theme.TEXT_LIGHT)
        entry = f"[{timestamp}] {message}"
        self.activity_log.append((entry, color))
        if len(self.activity_log) > self.max_log_entries:
            self.activity_log.pop(0)
        self._update_log_display()
        
    def _update_log_display(self):
        if self.log_listbox is not None:
            try:
                self.log_listbox.delete(0, tk.END)
                for entry, color in self.activity_log:
                    self.log_listbox.insert(tk.END, entry)
                    idx = self.log_listbox.size() - 1
                    self.log_listbox.itemconfig(idx, fg=color)
                if self.activity_log:
                    self.log_listbox.see(tk.END)
            except:
                pass
                
    def _clear_log(self):
        self.activity_log = []
        if self.log_listbox is not None:
            try:
                self.log_listbox.delete(0, tk.END)
            except:
                pass
        self._add_log("🗑️ لاگ‌ها پاک شدند", "warning")
    
    # ===========================
    # کلیدهای میانبر
    # ===========================
    
    def _setup_hotkeys(self):
        self.root.bind('<F9>', lambda e: self._toggle_monitoring())
        self.root.bind('<F10>', lambda e: self._save_data(save_to_history=True))
        self.root.bind('<F11>', lambda e: self._show_history())
        self.root.bind('<Escape>', lambda e: self._on_closing())
        
    def _setup_protocols(self):
        self.root.protocol("WM_DELETE_WINDOW", self._on_closing)
        
    def _toggle_monitoring(self):
        if self.is_monitoring:
            self._stop_monitoring()
        else:
            self._start_monitoring()
    
    # ===========================
    # راه‌اندازی UI
    # ===========================
    
    def _setup_ui(self):
        # ===== هدر =====
        header = tk.Frame(self.root, bg=Theme.HEADER, height=65)
        header.pack(fill='x')
        header.pack_propagate(False)
        
        title_frame = tk.Frame(header, bg=Theme.HEADER)
        title_frame.pack(side='left', padx=20, pady=10)
        
        tk.Label(title_frame, text="🧠 NeuroHabit", 
                font=Theme.FONT_TITLE,
                bg=Theme.HEADER, fg=Theme.PRIMARY).pack(side='left')
        
        tk.Label(title_frame, text="Pro 5.0", 
                font=('Segoe UI', 10, 'bold'),
                bg=Theme.HEADER, fg=Theme.GOLD).pack(side='left', padx=6)
        
        self.status_label = tk.Label(header, text="⏸️ متوقف", 
                                    font=Theme.FONT_HEADING,
                                    bg=Theme.HEADER, fg=Theme.TEXT_LIGHT)
        self.status_label.pack(side='right', padx=20, pady=10)
        
        # ===== نوار ابزار =====
        toolbar = tk.Frame(self.root, bg=Theme.BG, height=45)
        toolbar.pack(fill='x')
        toolbar.pack_propagate(False)
        
        btn_style = Theme.BUTTON_STYLE
        
        self.start_btn = tk.Button(toolbar, text="▶️ شروع", 
                                  bg=Theme.SUCCESS, fg='white',
                                  command=self._start_monitoring, **btn_style)
        self.start_btn.pack(side='left', padx=5, pady=4)
        
        self.stop_btn = tk.Button(toolbar, text="⏹️ توقف", 
                                 bg=Theme.DANGER, fg='white',
                                 command=self._stop_monitoring, 
                                 state='disabled', **btn_style)
        self.stop_btn.pack(side='left', padx=3, pady=4)
        
        self.save_btn = tk.Button(toolbar, text="💾 ذخیره جلسه", 
                                 bg=Theme.INFO, fg='white',
                                 command=lambda: self._save_data(save_to_history=True), 
                                 **btn_style)
        self.save_btn.pack(side='left', padx=3, pady=4)
        
        self.history_btn = tk.Button(toolbar, text="📜 تاریخچه", 
                                    bg=Theme.PRIMARY, fg='white',
                                    command=self._show_history, **btn_style)
        self.history_btn.pack(side='left', padx=3, pady=4)
        
        self.pdf_btn = tk.Button(toolbar, text="📄 PDF", 
                                bg=Theme.PINK, fg='white',
                                command=self._export_pdf, **btn_style)
        self.pdf_btn.pack(side='left', padx=3, pady=4)
        
        self.clear_btn = tk.Button(toolbar, text="🗑️ پاک کردن", 
                                  bg=Theme.DANGER, fg='white',
                                  command=self._clear_all_data, **btn_style)
        self.clear_btn.pack(side='left', padx=3, pady=4)
        
        self.auto_save_var = tk.BooleanVar(value=True)
        auto_switch = tk.Checkbutton(toolbar, text="🔄 خودکار", 
                                    variable=self.auto_save_var,
                                    bg=Theme.BG, fg=Theme.TEXT,
                                    selectcolor=Theme.BG,
                                    font=Theme.FONT_SMALL,
                                    command=self._toggle_auto_save)
        auto_switch.pack(side='left', padx=10, pady=4)
        
        # ===== کارت‌های آمار لحظه‌ای =====
        stats_frame = tk.Frame(self.root, bg=Theme.BG)
        stats_frame.pack(fill='x', padx=15, pady=5)
        
        card_data = [
            ('⌨️ کلیدها', 'keys', Theme.PRIMARY),
            ('🖱️ کلیک‌ها', 'clicks', Theme.SUCCESS),
            ('💨 سرعت', 'wpm', Theme.WARNING),
            ('⏱️ زمان', 'time', Theme.INFO)
        ]
        
        for label, key, color in card_data:
            card = tk.Frame(stats_frame, bg=Theme.CARD, 
                           highlightthickness=1,
                           highlightbackground=color,
                           width=200, height=70)
            card.pack(side='left', padx=4, fill='x', expand=True)
            card.pack_propagate(False)
            
            tk.Label(card, text=label, font=Theme.FONT_SMALL,
                    bg=Theme.CARD, fg=Theme.TEXT_LIGHT).pack(
                        anchor='w', padx=15, pady=(5,0))
            
            value = tk.Label(card, text="۰", font=Theme.FONT_LARGE,
                            bg=Theme.CARD, fg=color)
            value.pack(anchor='w', padx=15, pady=(0,5))
            
            self.live_cards[key] = value
        
        # ===== بخش تایمر استراحت =====
        reminder_frame = tk.Frame(self.root, bg=Theme.BG)
        reminder_frame.pack(fill='x', padx=15, pady=5)
        
        reminder_header = tk.Frame(reminder_frame, bg=Theme.BG)
        reminder_header.pack(fill='x')
        
        tk.Label(reminder_header, text="⏱️ تایمر استراحت", 
                font=Theme.FONT_HEADING,
                bg=Theme.BG, fg=Theme.INFO).pack(side='left', padx=5)
        
        self.reminder_label = tk.Label(reminder_header, text="⏱️ ۰۰:۰۰", 
                                      font=Theme.FONT_HEADING,
                                      bg=Theme.BG, fg=Theme.GOLD)
        self.reminder_label.pack(side='right', padx=5)
        
        reminder_controls = tk.Frame(reminder_frame, bg=Theme.BG)
        reminder_controls.pack(fill='x', pady=3)
        
        tk.Label(reminder_controls, text="هر", font=Theme.FONT_BODY,
                bg=Theme.BG, fg=Theme.TEXT).pack(side='left', padx=5)
        
        self.interval_var = tk.StringVar(value="40")
        interval_spin = tk.Spinbox(reminder_controls, from_=5, to=120, 
                                  textvariable=self.interval_var,
                                  width=5, font=Theme.FONT_BODY,
                                  bg=Theme.CARD, fg=Theme.TEXT,
                                  relief='flat', bd=1)
        interval_spin.pack(side='left', padx=2)
        
        tk.Label(reminder_controls, text="دقیقه", font=Theme.FONT_BODY,
                bg=Theme.BG, fg=Theme.TEXT).pack(side='left', padx=2)
        
        self.reminder_start_btn = tk.Button(reminder_controls, text="▶️ شروع", 
                                           bg=Theme.SUCCESS, fg='white',
                                           font=Theme.FONT_SMALL,
                                           relief='flat', bd=0, cursor='hand2',
                                           padx=10, pady=3,
                                           command=self._start_reminder)
        self.reminder_start_btn.pack(side='left', padx=5)
        
        self.reminder_stop_btn = tk.Button(reminder_controls, text="⏹️ توقف", 
                                          bg=Theme.DANGER, fg='white',
                                          font=Theme.FONT_SMALL,
                                          relief='flat', bd=0, cursor='hand2',
                                          padx=10, pady=3,
                                          command=self._stop_reminder,
                                          state='disabled')
        self.reminder_stop_btn.pack(side='left', padx=2)
        
        self.reminder_sound_btn = tk.Button(reminder_controls, text="🔊 انتخاب صدا", 
                                           bg=Theme.PRIMARY, fg='white',
                                           font=Theme.FONT_SMALL,
                                           relief='flat', bd=0, cursor='hand2',
                                           padx=10, pady=3,
                                           command=self._choose_sound)
        self.reminder_sound_btn.pack(side='left', padx=2)
        
        self.sound_status_label = tk.Label(reminder_controls, text="🔇", 
                                          font=Theme.FONT_SMALL,
                                          bg=Theme.BG, fg=Theme.TEXT_LIGHT)
        self.sound_status_label.pack(side='left', padx=5)
        
        # ===== فریم اصلی =====
        main_frame = tk.Frame(self.root, bg=Theme.BG)
        main_frame.pack(fill='both', expand=True, padx=10, pady=5)
        
        # ===== پنل سمت چپ =====
        left_panel = tk.Frame(main_frame, bg=Theme.BG, width=450)
        left_panel.pack(side='left', fill='both', expand=True)
        left_panel.pack_propagate(False)
        
        # بخش جلسه فعلی
        tk.Label(left_panel, text="📊 جلسه فعلی", 
                font=Theme.FONT_HEADING,
                bg=Theme.BG, fg=Theme.PRIMARY).pack(anchor='w', padx=5, pady=(5,2))
        
        self.session_frame = tk.Frame(left_panel, bg=Theme.CARD, 
                                     highlightthickness=1,
                                     highlightbackground=Theme.BORDER)
        self.session_frame.pack(fill='both', expand=True, padx=3, pady=2)
        
        # ===== Activity Log =====
        log_label = tk.Frame(left_panel, bg=Theme.BG)
        log_label.pack(fill='x', pady=(5,2))
        
        tk.Label(log_label, text="📋 گزارش لحظه‌ای", 
                font=Theme.FONT_HEADING,
                bg=Theme.BG, fg=Theme.INFO).pack(side='left', padx=5)
        
        clear_log_btn = tk.Button(log_label, text="🗑️ پاک کردن لاگ", 
                                 font=Theme.FONT_SMALL,
                                 bg=Theme.BG, fg=Theme.TEXT_LIGHT,
                                 relief='flat', bd=0, cursor='hand2',
                                 command=self._clear_log)
        clear_log_btn.pack(side='right', padx=5)
        
        log_frame = tk.Frame(left_panel, bg=Theme.CARD,
                            highlightthickness=1,
                            highlightbackground=Theme.BORDER,
                            height=110)
        log_frame.pack(fill='x', padx=3, pady=2)
        log_frame.pack_propagate(False)
        
        self.log_listbox = tk.Listbox(log_frame, 
                                      bg=Theme.CARD, 
                                      fg=Theme.TEXT,
                                      font=Theme.FONT_MONO,
                                      relief='flat',
                                      bd=0,
                                      highlightthickness=0,
                                      selectbackground=Theme.PRIMARY,
                                      height=4)
        self.log_listbox.pack(fill='both', expand=True, padx=5, pady=5)
        
        log_scrollbar = tk.Scrollbar(self.log_listbox)
        log_scrollbar.pack(side='right', fill='y')
        self.log_listbox.config(yscrollcommand=log_scrollbar.set)
        log_scrollbar.config(command=self.log_listbox.yview)
        
        # ===== پنل سمت راست (تاریخچه) =====
        right_panel = tk.Frame(main_frame, bg=Theme.BG, width=450)
        right_panel.pack(side='right', fill='both', expand=True)
        right_panel.pack_propagate(False)
        
        history_header = tk.Frame(right_panel, bg=Theme.BG)
        history_header.pack(fill='x', pady=(5,2))
        
        tk.Label(history_header, text="📜 تاریخچه جلسات", 
                font=Theme.FONT_HEADING,
                bg=Theme.BG, fg=Theme.GOLD).pack(side='left', padx=5)
        
        tk.Label(history_header, text=f"{len(self.history_data)} جلسه", 
                font=Theme.FONT_SMALL,
                bg=Theme.BG, fg=Theme.TEXT_LIGHT).pack(side='right', padx=5)
        
        self.history_frame = tk.Frame(right_panel, bg=Theme.CARD,
                                      highlightthickness=1,
                                      highlightbackground=Theme.BORDER)
        self.history_frame.pack(fill='both', expand=True, padx=3, pady=2)
        
        # ===== فوتر =====
        footer = tk.Frame(self.root, bg=Theme.HEADER, height=28)
        footer.pack(side='bottom', fill='x')
        footer.pack_propagate(False)
        
        self.footer_label = tk.Label(footer, 
            text="✅ آماده | F9: شروع/توقف | F10: ذخیره جلسه | F11: تاریخچه | Esc: خروج", 
            font=Theme.FONT_SMALL,
            bg=Theme.HEADER, fg=Theme.TEXT_LIGHT)
        self.footer_label.pack(side='left', padx=15)
        
        self.timer_label = tk.Label(footer, text="⏱️ ۰۰:۰۰", 
                                   font=Theme.FONT_HEADING,
                                   bg=Theme.HEADER, fg=Theme.INFO)
        self.timer_label.pack(side='right', padx=15)
        
        # ===== نمایش اولیه =====
        self._show_session_info(None)
        self._show_history_list()
        self._add_log("📂 رابط کاربری آماده شد", "system")
    
    # ===========================
    # توابع Smart Reminder
    # ===========================
    
    def _start_reminder(self):
        try:
            interval = int(self.interval_var.get())
            if interval < 5:
                messagebox.showwarning("هشدار", "حداقل زمان ۵ دقیقه است!")
                return
                
            self.reminder = SmartReminder(self)
            self.reminder.set_interval(interval)
            self.reminder.start()
            
            self.reminder_start_btn.config(state='disabled')
            self.reminder_stop_btn.config(state='normal')
            self._add_log(f"⏱️ تایمر استراحت شروع شد ({interval} دقیقه)", "monitor")
            
        except Exception as e:
            messagebox.showerror("خطا", f"خطا در شروع تایمر:\n{e}")

    def _stop_reminder(self):
        if hasattr(self, 'reminder') and self.reminder:
            self.reminder.stop()
            self.reminder_start_btn.config(state='normal')
            self.reminder_stop_btn.config(state='disabled')
            self.reminder_label.config(text="⏱️ ۰۰:۰۰")
            self._add_log("⏹️ تایمر استراحت متوقف شد", "monitor")

    def _choose_sound(self):
        file_path = filedialog.askopenfilename(
            title="انتخاب صدای هشدار",
            filetypes=[
                ("Audio files", "*.wav *.mp3 *.ogg"),
                ("All files", "*.*")
            ]
        )
        if file_path:
            if hasattr(self, 'reminder') and self.reminder:
                if self.reminder.set_sound(file_path):
                    self.sound_status_label.config(text="🔊", fg=Theme.SUCCESS)
                    self._add_log(f"🔊 صدای جدید انتخاب شد: {os.path.basename(file_path)}", "save")
                else:
                    messagebox.showerror("خطا", "فایل صوتی معتبر نیست!")
    
    # ===========================
    # نمایش اطلاعات جلسه
    # ===========================
    
    def _show_session_info(self, data):
        for widget in self.session_frame.winfo_children():
            widget.destroy()
        
        if data:
            keyboard = data.get('keyboard', {})
            mouse = data.get('mouse', {})
            app_report = data.get('app_report', {})
            score = data.get('productivity_score', 0)
            
            # ===== کارت‌های اصلی =====
            info_frame = tk.Frame(self.session_frame, bg=Theme.CARD)
            info_frame.pack(fill='x', padx=10, pady=8)
            
            score_color = self._get_score_color(score)
            items = [
                ('🎯 بهره‌وری', f"{score:.0f}%", score_color),
                ('⌨️ کلیدها', f"{keyboard.get('total_keys', 0):,}", Theme.PRIMARY),
                ('🖱️ کلیک‌ها', f"{mouse.get('total_clicks', 0):,}", Theme.SUCCESS),
                ('⏱️ مدت', f"{keyboard.get('duration_minutes', 0):.0f}m", Theme.INFO)
            ]
            
            for label, value, color in items:
                frame = tk.Frame(info_frame, bg=Theme.CARD)
                frame.pack(fill='x', pady=2)
                tk.Label(frame, text=label, font=Theme.FONT_SMALL,
                        bg=Theme.CARD, fg=Theme.TEXT_LIGHT).pack(side='left', padx=5)
                tk.Label(frame, text=value, font=Theme.FONT_HEADING,
                        bg=Theme.CARD, fg=color).pack(side='right', padx=5)
            
            # ===== جزئیات =====
            detail_frame = tk.Frame(self.session_frame, bg=Theme.CARD)
            detail_frame.pack(fill='x', padx=10, pady=5)
            
            kb_text = f"⌨️ سرعت: {keyboard.get('typing_speed_wpm', 0):.1f} WPM | Backspace: {keyboard.get('backspaces', 0)}"
            tk.Label(detail_frame, text=kb_text, font=Theme.FONT_SMALL,
                    bg=Theme.CARD, fg=Theme.TEXT).pack(anchor='w', pady=1)
            
            ms_text = f"🖱️ مسافت: {mouse.get('total_distance', 0):.0f}px | حرکت: {mouse.get('movement_count', 0)}"
            tk.Label(detail_frame, text=ms_text, font=Theme.FONT_SMALL,
                    bg=Theme.CARD, fg=Theme.TEXT).pack(anchor='w', pady=1)
            
            # ===== AppMonitor =====
            if app_report and app_report.get('total_apps', 0) > 0:
                app_frame = tk.Frame(self.session_frame, bg=Theme.CARD)
                app_frame.pack(fill='x', padx=10, pady=5)
                
                tk.Label(app_frame, text="📱 برنامه‌های پرکاربرد", 
                        font=Theme.FONT_HEADING,
                        bg=Theme.CARD, fg=Theme.CYAN).pack(anchor='w')
                
                for i, (app, minutes) in enumerate(list(app_report.get('app_usage', {}).items())[:3], 1):
                    tk.Label(app_frame, text=f"   {i}. {app}: {minutes:.1f} دقیقه", 
                            font=Theme.FONT_SMALL,
                            bg=Theme.CARD, fg=Theme.TEXT).pack(anchor='w', padx=10)
                
                # ===== گزارش حواس‌پرتی (ممد) =====
                productive_time = app_report.get('productive_time', 0)
                distraction_time = app_report.get('distraction_time', 0)
                total_time = productive_time + distraction_time
                
                if total_time > 0:
                    ratio = (distraction_time / total_time) * 100
                    if ratio < 20:
                        distract_status = "🔥 کمترین حواس‌پرتی"
                        distract_color = Theme.SUCCESS
                    elif ratio < 40:
                        distract_status = "💪 حواس‌پرتی قابل قبول"
                        distract_color = Theme.WARNING
                    elif ratio < 60:
                        distract_status = "📈 حواس‌پرتی متوسط"
                        distract_color = '#e67e22'
                    else:
                        distract_status = "😴 حواس‌پرتی زیاد!"
                        distract_color = Theme.DANGER
                    
                    tk.Label(app_frame, text=f"⚠️ حواس‌پرتی: {distract_status} ({ratio:.0f}%)", 
                            font=Theme.FONT_SMALL,
                            bg=Theme.CARD, fg=distract_color).pack(anchor='w', padx=10)
                
                # بهره‌وری برنامه‌ها
                ratio = app_report.get('productivity_ratio', 0)
                status = app_report.get('status', '')
                color = Theme.SUCCESS if ratio >= 60 else Theme.WARNING if ratio >= 40 else Theme.DANGER
                tk.Label(app_frame, text=f"🎯 بهره‌وری برنامه‌ها: {ratio:.0f}% - {status}", 
                        font=Theme.FONT_SMALL,
                        bg=Theme.CARD, fg=color).pack(anchor='w', padx=10)
            
            # ===== وضعیت =====
            status_text = self._get_status_text(score)
            status_frame = tk.Frame(self.session_frame, bg=Theme.CARD)
            status_frame.pack(fill='x', padx=10, pady=8)
            tk.Label(status_frame, text=f"💡 {status_text}", 
                    font=Theme.FONT_SMALL,
                    bg=Theme.CARD, fg=Theme.WARNING).pack()
        else:
            empty_frame = tk.Frame(self.session_frame, bg=Theme.CARD)
            empty_frame.pack(fill='both', expand=True)
            tk.Label(empty_frame, text="📭", font=('Segoe UI', 32),
                    bg=Theme.CARD, fg=Theme.TEXT_LIGHT).pack(expand=True)
            tk.Label(empty_frame, text="هنوز جلسه‌ای ذخیره نشده", 
                    font=Theme.FONT_BODY, bg=Theme.CARD, fg=Theme.TEXT_LIGHT).pack()
    
    # ===========================
    # نمایش تاریخچه
    # ===========================
    
    def _show_history_list(self):
        for widget in self.history_frame.winfo_children():
            widget.destroy()
        
        if not self.history_data:
            empty_frame = tk.Frame(self.history_frame, bg=Theme.CARD)
            empty_frame.pack(fill='both', expand=True)
            tk.Label(empty_frame, text="📭", font=('Segoe UI', 32),
                    bg=Theme.CARD, fg=Theme.TEXT_LIGHT).pack(expand=True)
            tk.Label(empty_frame, text="هیچ جلسه‌ای ذخیره نشده", 
                    font=Theme.FONT_BODY, bg=Theme.CARD, fg=Theme.TEXT_LIGHT).pack()
            return
        
        canvas = tk.Canvas(self.history_frame, bg=Theme.CARD, highlightthickness=0)
        scrollbar = tk.Scrollbar(self.history_frame, orient="vertical", command=canvas.yview)
        scrollable_frame = tk.Frame(canvas, bg=Theme.CARD)
        
        scrollable_frame.bind(
            "<Configure>",
            lambda e: canvas.configure(scrollregion=canvas.bbox("all"))
        )
        
        canvas.create_window((0, 0), window=scrollable_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)
        
        for data in self.history_data[:10]:
            keyboard = data.get('keyboard', {})
            mouse = data.get('mouse', {})
            score = data.get('productivity_score', 0)
            app_report = data.get('app_report', {})
            timestamp = data.get('timestamp', '')
            
            time_str = ""
            if timestamp:
                try:
                    dt = datetime.fromisoformat(timestamp)
                    time_str = dt.strftime("%H:%M")
                except:
                    pass
            
            card = tk.Frame(scrollable_frame, bg=Theme.CARD, 
                          highlightthickness=1,
                          highlightbackground=Theme.BORDER)
            card.pack(fill='x', pady=2, padx=3)
            
            row1 = tk.Frame(card, bg=Theme.CARD)
            row1.pack(fill='x', padx=8, pady=2)
            
            score_color = self._get_score_color(score)
            tk.Label(row1, text=f"🎯 {score:.0f}%", 
                    font=Theme.FONT_HEADING,
                    bg=Theme.CARD, fg=score_color).pack(side='left')
            tk.Label(row1, text=f"🕐 {time_str}", 
                    font=Theme.FONT_SMALL,
                    bg=Theme.CARD, fg=Theme.TEXT_LIGHT).pack(side='right')
            
            row2 = tk.Frame(card, bg=Theme.CARD)
            row2.pack(fill='x', padx=8, pady=2)
            
            tk.Label(row2, text=f"⌨️ {keyboard.get('total_keys', 0)}  🖱️ {mouse.get('total_clicks', 0)}  ⏱️ {keyboard.get('duration_minutes', 0):.0f}m", 
                    font=Theme.FONT_SMALL,
                    bg=Theme.CARD, fg=Theme.TEXT_LIGHT).pack(side='left')
            
            view_btn = tk.Button(row2, text="👁️ مشاهده", 
                                font=Theme.FONT_SMALL,
                                bg=Theme.CARD, fg=Theme.PRIMARY,
                                relief='flat', bd=0, cursor='hand2',
                                command=lambda d=data: self._show_session_info(d))
            view_btn.pack(side='right')
            
            # AppMonitor + حواس‌پرتی
            if app_report and app_report.get('total_apps', 0) > 0:
                row3 = tk.Frame(card, bg=Theme.CARD)
                row3.pack(fill='x', padx=8, pady=2)
                
                top_app = app_report.get('top_app', 'هیچ')
                ratio = app_report.get('productivity_ratio', 0)
                
                # حواس‌پرتی
                productive_time = app_report.get('productive_time', 0)
                distraction_time = app_report.get('distraction_time', 0)
                total_time = productive_time + distraction_time
                distract_ratio = (distraction_time / total_time * 100) if total_time > 0 else 0
                
                if distract_ratio < 30:
                    emoji = "🔥"
                elif distract_ratio < 50:
                    emoji = "💪"
                elif distract_ratio < 70:
                    emoji = "📈"
                else:
                    emoji = "😴"
                
                tk.Label(row3, text=f"📱 {top_app} | 🎯 {ratio:.0f}% | {emoji} حواس‌پرتی {distract_ratio:.0f}%", 
                        font=Theme.FONT_SMALL,
                        bg=Theme.CARD, fg=Theme.CYAN).pack(side='left')
        
        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")
    
    def _show_history(self):
        self._load_history()
        self._show_history_list()
        self._add_log(f"📜 تاریخچه بروزرسانی شد ({len(self.history_data)} جلسه)", "system")
        messagebox.showinfo("تاریخچه", f"📜 {len(self.history_data)} جلسه ذخیره شده است.")
    
    # ===========================
    # خروجی PDF
    # ===========================
    
    def _export_pdf(self):
        if not self.history_data:
            messagebox.showwarning("هشدار", "هیچ داده‌ای برای خروجی وجود ندارد!")
            return
        
        file_path = filedialog.asksaveasfilename(
            defaultextension=".txt",
            filetypes=[("Text files", "*.txt"), ("All files", "*.*")],
            title="ذخیره گزارش"
        )
        
        if not file_path:
            return
        
        try:
            with open(file_path, 'w', encoding='utf-8') as f:
                f.write("=" * 60 + "\n")
                f.write("🧠 NeuroHabit - گزارش تاریخچه\n")
                f.write("=" * 60 + "\n\n")
                f.write(f"تاریخ: {datetime.now().strftime('%Y-%m-%d %H:%M')}\n")
                f.write(f"تعداد جلسات: {len(self.history_data)}\n\n")
                f.write("-" * 60 + "\n\n")
                
                for i, data in enumerate(self.history_data[:20], 1):
                    keyboard = data.get('keyboard', {})
                    mouse = data.get('mouse', {})
                    score = data.get('productivity_score', 0)
                    app_report = data.get('app_report', {})
                    timestamp = data.get('timestamp', '')
                    
                    f.write(f"📊 جلسه {i}\n")
                    f.write(f"   زمان: {timestamp}\n")
                    f.write(f"   بهره‌وری: {score:.0f}%\n")
                    f.write(f"   کلیدها: {keyboard.get('total_keys', 0)}\n")
                    f.write(f"   کلیک‌ها: {mouse.get('total_clicks', 0)}\n")
                    f.write(f"   مدت زمان: {keyboard.get('duration_minutes', 0):.0f} دقیقه\n")
                    f.write(f"   سرعت تایپ: {keyboard.get('typing_speed_wpm', 0):.1f} WPM\n")
                    f.write(f"   Backspace: {keyboard.get('backspaces', 0)}\n")
                    f.write(f"   مسافت ماوس: {mouse.get('total_distance', 0):.0f} پیکسل\n")
                    f.write(f"   حرکت بی‌قرار: {mouse.get('restless_moves', 0)}\n")
                    
                    if app_report and app_report.get('total_apps', 0) > 0:
                        productive_time = app_report.get('productive_time', 0)
                        distraction_time = app_report.get('distraction_time', 0)
                        total_time = productive_time + distraction_time
                        distract_ratio = (distraction_time / total_time * 100) if total_time > 0 else 0
                        
                        f.write(f"   📱 برنامه برتر: {app_report.get('top_app', 'هیچ')}\n")
                        f.write(f"   🎯 بهره‌وری برنامه‌ها: {app_report.get('productivity_ratio', 0):.0f}%\n")
                        f.write(f"   ⚠️ حواس‌پرتی: {distract_ratio:.0f}%\n")
                    
                    f.write("-" * 40 + "\n\n")
                
                f.write("=" * 60 + "\n")
                f.write("پایان گزارش\n")
            
            self._add_log(f"📄 گزارش PDF ذخیره شد: {os.path.basename(file_path)}", "save")
            messagebox.showinfo("موفق", f"✅ گزارش با موفقیت ذخیره شد!\n\n{file_path}")
            
        except Exception as e:
            messagebox.showerror("خطا", f"خطا در ذخیره گزارش:\n{e}")
    
    # ===========================
    # توابع کمکی
    # ===========================
    
    def _get_score_color(self, score):
        if score >= 80:
            return Theme.SUCCESS
        elif score >= 60:
            return Theme.WARNING
        elif score >= 40:
            return '#e67e22'
        else:
            return Theme.DANGER
            
    def _get_status_text(self, score):
        if score >= 80:
            return "🔥 عالی! در اوج بهره‌وری هستی!"
        elif score >= 60:
            return "💪 خوب! می‌تونی بهتر بشی!"
        elif score >= 40:
            return "📈 قابل قبول! نیاز به بهبود داری!"
        else:
            return "😴 امروز روزت نیست! استراحت کن!"
            
    def _calculate_score(self, keyboard, mouse):
        score = 0
        
        wpm = keyboard.get('typing_speed_wpm', 0)
        if wpm >= 40:
            score += 30
        elif wpm >= 30:
            score += 25
        elif wpm >= 20:
            score += 15
        elif wpm >= 10:
            score += 10
        else:
            score += 5
            
        keys = keyboard.get('total_keys', 0)
        if keys >= 100:
            score += 20
        elif keys >= 50:
            score += 15
        elif keys >= 20:
            score += 10
        else:
            score += 5
            
        clicks = mouse.get('total_clicks', 0)
        if clicks >= 50:
            score += 20
        elif clicks >= 30:
            score += 15
        elif clicks >= 10:
            score += 10
        else:
            score += 5
            
        restless = mouse.get('restless_moves', 0)
        if restless <= 2:
            score += 15
        elif restless <= 5:
            score += 10
        elif restless <= 10:
            score += 5
        else:
            score += 0
            
        duration = keyboard.get('duration_minutes', 0)
        if duration >= 10:
            score += 15
        elif duration >= 5:
            score += 10
        elif duration >= 2:
            score += 5
        else:
            score += 0
            
        return min(100, score)
    
    # ===========================
    # توابع اصلی
    # ===========================
    
    def _start_monitoring(self):
        if self.is_monitoring:
            return
            
        try:
            self.keyboard.start()
            self.mouse.start()
            self.app_monitor.start()
            self.is_monitoring = True
            self.session_start = time.time()
            
            self.start_btn.config(state='disabled')
            self.stop_btn.config(state='normal')
            self.status_label.config(text="🟢 در حال مانیتورینگ...", fg=Theme.SUCCESS)
            self.footer_label.config(text="✅ در حال ضبط... | F9: توقف")
            
            if self.auto_save_var.get():
                self._start_auto_save()
                
            self._update_live_stats()
            
            self._add_log("▶️ مانیتورینگ شروع شد", "monitor")
            self._add_log("⌨️ کیبورد و 🖱️ ماوس فعال شدند", "success")
            
            messagebox.showinfo("NeuroHabit", "✅ مانیتورینگ شروع شد!")
                
        except Exception as e:
            messagebox.showerror("خطا", f"خطا در شروع:\n{e}")
            
    def _stop_monitoring(self):
        if not self.is_monitoring:
            return
            
        try:
            self.keyboard.stop()
            self.mouse.stop()
            self.app_monitor.stop()
            self.is_monitoring = False
            
            self.start_btn.config(state='normal')
            self.stop_btn.config(state='disabled')
            self.status_label.config(text="⏸️ متوقف", fg=Theme.TEXT_LIGHT)
            self.footer_label.config(text="⏸️ متوقف | F9: شروع")
            
            self._stop_auto_save()
            
            # ذخیره نهایی
            kb_report = self.keyboard.get_report()
            ms_report = self.mouse.get_report()
            app_report = self.app_monitor.get_report()
            self._save_data_with_reports(kb_report, ms_report, app_report, save_to_history=True)
            
            self._add_log("⏹️ مانیتورینگ متوقف شد", "monitor")
            self._add_log("💾 جلسه در تاریخچه ثبت شد", "save")
            
            messagebox.showinfo("NeuroHabit", "⏹️ مانیتورینگ متوقف شد.\nجلسه در تاریخچه ثبت شد.")
            
        except Exception as e:
            messagebox.showerror("خطا", f"خطا در توقف:\n{e}")
            
    def _save_data(self, save_to_history=True):
        if not self.is_monitoring:
            messagebox.showwarning("هشدار", "ابتدا مانیتورینگ را شروع کنید!")
            return
            
        try:
            kb = self.keyboard.get_report()
            ms = self.mouse.get_report()
            app_report = self.app_monitor.get_report()
            self._save_data_with_reports(kb, ms, app_report, save_to_history)
            
        except Exception as e:
            messagebox.showerror("خطا", f"خطا در ذخیره:\n{e}")
            
    def _save_data_with_reports(self, kb, ms, app_report, save_to_history=True):
        """ذخیره داده با گزارش‌های دریافتی"""
        try:
            self.logger.save_keyboard_data(kb)
            self.logger.save_mouse_data(ms)
            
            if save_to_history:
                self.logger.save_combined_data(kb, ms, app_report)
                
                score = self._calculate_score(kb, ms)
                
                self.current_session_data = {
                    'keyboard': kb,
                    'mouse': ms,
                    'app_report': app_report,
                    'productivity_score': score,
                    'timestamp': datetime.now().isoformat()
                }
                
                self._load_history()
                self._show_session_info(self.current_session_data)
                self._show_history_list()
                
                self._add_log(f"💾 جلسه جدید ذخیره شد | کلیدها: {kb.get('total_keys', 0)}", "save")
                self.status_label.config(text="💾 ذخیره شد!", fg=Theme.INFO)
            else:
                self._add_log(f"🔄 به‌روزرسانی خودکار | کلیدها: {kb.get('total_keys', 0)}", "info")
                self.status_label.config(text="🔄 به‌روز شد", fg=Theme.TEXT_LIGHT)
            
            now = datetime.now().strftime("%H:%M:%S")
            self.footer_label.config(text=f"💾 {now}")
            
        except Exception as e:
            print(f"Error in save_data_with_reports: {e}")
            raise
            
    def _clear_all_data(self):
        if messagebox.askyesno("تأیید", "همه داده‌ها پاک می‌شوند!\nآیا مطمئن هستید؟"):
            try:
                import shutil
                if os.path.exists("data"):
                    shutil.rmtree("data")
                    os.makedirs("data/raw")
                    os.makedirs("data/summary")
                    os.makedirs("data/logs")
                
                self.history_data = []
                self.current_session_data = None
                self._show_session_info(None)
                self._show_history_list()
                self._add_log("🗑️ همه داده‌ها پاک شدند", "error")
                messagebox.showinfo("موفق", "✅ همه داده‌ها پاک شدند!")
            except Exception as e:
                messagebox.showerror("خطا", f"خطا در پاک کردن:\n{e}")
    
    # ===========================
    # ذخیره خودکار
    # ===========================
    
    def _start_auto_save(self):
        if self.is_auto_save_running:
            return
        self.is_auto_save_running = True
        
        def auto_save_loop():
            while self.is_auto_save_running and self.is_monitoring:
                time.sleep(self.auto_save_interval)
                if self.is_monitoring:
                    try:
                        self.root.after(0, lambda: self._save_data(save_to_history=False))
                    except:
                        pass
                    
        self.auto_save_thread = threading.Thread(target=auto_save_loop, daemon=True)
        self.auto_save_thread.start()
        
    def _stop_auto_save(self):
        self.is_auto_save_running = False
        
    def _toggle_auto_save(self):
        if self.auto_save_var.get():
            if self.is_monitoring:
                self._start_auto_save()
        else:
            self._stop_auto_save()
    
    # ===========================
    # آمار لحظه‌ای
    # ===========================
    
    def _update_live_stats(self):
        if not self.is_monitoring:
            return
            
        try:
            kb = self.keyboard.get_report()
            ms = self.mouse.get_report()
            
            self.live_cards['keys'].config(text=str(kb.get('total_keys', 0)))
            self.live_cards['clicks'].config(text=str(ms.get('total_clicks', 0)))
            self.live_cards['wpm'].config(text=f"{kb.get('typing_speed_wpm', 0):.1f}")
            
            if self.session_start:
                elapsed = int(time.time() - self.session_start)
                mins = elapsed // 60
                secs = elapsed % 60
                self.live_cards['time'].config(text=f"{mins:02d}:{secs:02d}")
                self.timer_label.config(text=f"⏱️ {mins:02d}:{secs:02d}")
            
            if int(time.time()) % 10 == 0:
                self._add_log(f"📊 کلیدها: {kb.get('total_keys', 0)} | WPM: {kb.get('typing_speed_wpm', 0):.0f}", "info")
            
            self.root.after(1000, self._update_live_stats)
            
        except Exception as e:
            self.root.after(2000, self._update_live_stats)
            
    def _update_timer(self):
        try:
            if self.session_start and self.is_monitoring:
                elapsed = int(time.time() - self.session_start)
                mins = elapsed // 60
                secs = elapsed % 60
                self.timer_label.config(text=f"⏱️ {mins:02d}:{secs:02d}")
        except:
            pass
        self.root.after(1000, self._update_timer)
    

def _update_display(self):
    """به‌روزرسانی نمایش تایمر (در Main Thread اجرا میشه)"""
    if hasattr(self.parent, 'reminder_label'):
        mins = self.remaining_time // 60
        secs = self.remaining_time % 60
        # ===== نمایش با فرمت دقیقه:ثانیه ====
        self.parent.reminder_label.config(text=f"⏱️ {mins:02d}:{secs:02d}")

    # ===========================
    # بستن برنامه
    # ===========================
    
    def _on_closing(self):
        try:
            if self.is_monitoring:
                kb_report = self.keyboard.get_report()
                ms_report = self.mouse.get_report()
                app_report = self.app_monitor.get_report()
                
                self.keyboard.stop()
                self.mouse.stop()
                self.app_monitor.stop()
                self.is_monitoring = False
                
                self._save_data_with_reports(kb_report, ms_report, app_report, save_to_history=True)
            
            if hasattr(self, 'reminder') and self.reminder:
                self.reminder.stop()
            
            self.root.destroy()
            
        except Exception as e:
            print(f"Error in shutdown: {e}")
            self.root.destroy()
    
    # ===========================
    # اجرای برنامه
    # ===========================
    
    def run(self):
        self.root.mainloop()

# ===========================
# اجرای اصلی
# ===========================

if __name__ == "__main__":
    try:
        app = NeuroHabitApp()
        app.run()
    except Exception as e:
        messagebox.showerror("خطای بحرانی", f"مشکلی در اجرا:\n\n{str(e)}")
        sys.exit(1)
