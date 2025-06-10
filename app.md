import tkinter as tk
from tkinter import ttk, messagebox, filedialog, simpledialog
import os
import sqlite3
import pandas as pd
from tkcalendar import DateEntry
import hashlib
import re
import json
import pyperclip
import zipfile
import tempfile
import shutil
import datetime
import hardware_verification
import uuid
import time
import openpyxl
import openpyxl.styles

try:
    from docx import Document
    from docx.shared import Pt, RGBColor, Inches
    from docx.enum.text import WD_ALIGN_PARAGRAPH
    from docx.enum.table import WD_ALIGN_VERTICAL, WD_TABLE_ALIGNMENT
    from docx.enum.section import WD_ORIENTATION
    from docx.oxml.ns import qn
    from docx.oxml.shared import parse_xml, nsdecls
    from docx.oxml import OxmlElement
except ImportError:
    print("تحذير: مكتبة python-docx غير مثبتة. قم بتثبيتها باستخدام: pip install python-docx")


try:
    import arabic_reshaper
    from bidi.algorithm import get_display


    def fix_arabic(text):
        reshaped = arabic_reshaper.reshape(text)
        return get_display(reshaped)
except ImportError:
    def fix_arabic(text):
        return text

try:
    from reportlab.pdfgen import canvas
    from reportlab.lib.pagesizes import A4, landscape
    from reportlab.pdfbase import pdfmetrics
    from reportlab.pdfbase.ttfonts import TTFont

    pdfmetrics.registerFont(TTFont('ArabicFont', 'Tajawal-Regular.ttf'))
except ImportError:
    print("تحذير: مكتبة reportlab غير مثبتة، وقد لا ينجح تصدير PDF.")


class LoginSystem:
    def __init__(self, root):
        self.root = root
        self.db_conn = self.connect_to_db()
        self.current_user = None
        self.create_users_table()
        self.create_permissions_table()
        self.check_admin_exists()
        self.setup_login_window()

    def connect_to_db(self):
        try:
            conn = sqlite3.connect("attendance.db")
            return conn
        except Exception as e:
            messagebox.showerror("خطأ في قاعدة البيانات", f"لا يمكن الاتصال بقاعدة البيانات: {str(e)}")
            exit(1)

    def create_users_table(self):
        try:
            with self.db_conn:
                try:
                    self.db_conn.execute("ALTER TABLE users DROP COLUMN role")
                except:
                    pass
                self.db_conn.execute("""
                    CREATE TABLE IF NOT EXISTS users (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        username TEXT UNIQUE,
                        password TEXT,
                        full_name TEXT,
                        created_date TEXT,
                        last_login TEXT,
                        is_active INTEGER DEFAULT 1
                    )
                """)
        except Exception as e:
            messagebox.showerror("خطأ", f"تعذّر إنشاء/تعديل جدول المستخدمين: {str(e)}")

    def create_permissions_table(self):
        try:
            with self.db_conn:
                # إنشاء الجدول أولاً
                self.db_conn.execute("""
                    CREATE TABLE IF NOT EXISTS user_permissions (
                        user_id INTEGER PRIMARY KEY,
                        can_edit_attendance INTEGER DEFAULT 1,
                        can_add_students INTEGER DEFAULT 1,
                        can_edit_students INTEGER DEFAULT 1,
                        can_delete_students INTEGER DEFAULT 0,
                        can_view_edit_history INTEGER DEFAULT 0,
                        can_reset_attendance INTEGER DEFAULT 0,
                        can_export_data INTEGER DEFAULT 1,
                        can_import_data INTEGER DEFAULT 0,
                        can_edit_old_attendance INTEGER DEFAULT 0,
                        is_admin INTEGER DEFAULT 0,
                        FOREIGN KEY (user_id) REFERENCES users(id)
                    )
                """)

                # بعد التأكد من وجود الجدول، نتحقق من الأعمدة
                cursor = self.db_conn.cursor()
                cursor.execute("PRAGMA table_info(user_permissions)")
                columns = [column[1] for column in cursor.fetchall()]

                # إضافة العمود الجديد إذا لم يكن موجوداً
                if columns and "can_edit_old_attendance" not in columns:
                    try:
                        self.db_conn.execute(
                            "ALTER TABLE user_permissions ADD COLUMN can_edit_old_attendance INTEGER DEFAULT 0"
                        )
                    except:
                        pass  # العمود موجود بالفعل

        except Exception as e:
            messagebox.showerror("خطأ", f"تعذّر إنشاء جدول الصلاحيات: {str(e)}")

    def check_admin_exists(self):
        cursor = self.db_conn.cursor()
        cursor.execute("SELECT COUNT(*) FROM users WHERE username='admin'")
        count = cursor.fetchone()[0]
        if count == 0:
            hashed_pwd = hashlib.sha256("admin123".encode()).hexdigest()
            try:
                with self.db_conn:
                    self.db_conn.execute("""
                        INSERT INTO users (username, password, full_name, created_date, is_active)
                        VALUES (?, ?, ?, ?, ?)
                    """, (
                        'admin',
                        hashed_pwd,
                        'المسؤول الرئيسي',
                        datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                        1
                    ))
                    cursor.execute("SELECT id FROM users WHERE username='admin'")
                    admin_id = cursor.fetchone()[0]

                    # تأكد من أن is_admin = 1 للمشرف
                    self.db_conn.execute("""
                        INSERT INTO user_permissions (
                            user_id, can_edit_attendance, can_add_students, 
                            can_edit_students, can_delete_students, can_view_edit_history,
                            can_reset_attendance, can_export_data, can_import_data, 
                            can_edit_old_attendance, is_admin
                        ) VALUES (?, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1)
                    """, (admin_id,))
            except Exception as e:
                messagebox.showerror("خطأ", f"تعذّر إنشاء حساب المدير الرئيسي: {str(e)}")
        else:
            # إذا كان المستخدم admin موجود، تأكد من أن صلاحياته صحيحة
            cursor.execute("SELECT id FROM users WHERE username='admin'")
            admin_id = cursor.fetchone()[0]

            # تحقق من وجود صلاحيات للمشرف
            cursor.execute("SELECT is_admin FROM user_permissions WHERE user_id=?", (admin_id,))
            result = cursor.fetchone()

            if not result or result[0] != 1:
                # إذا لم تكن الصلاحيات موجودة أو غير صحيحة، قم بتحديثها
                with self.db_conn:
                    # حذف الصلاحيات القديمة إن وجدت
                    self.db_conn.execute("DELETE FROM user_permissions WHERE user_id=?", (admin_id,))

                    # إضافة صلاحيات المشرف الصحيحة
                    self.db_conn.execute("""
                        INSERT INTO user_permissions (
                            user_id, can_edit_attendance, can_add_students, 
                            can_edit_students, can_delete_students, can_view_edit_history,
                            can_reset_attendance, can_export_data, can_import_data, 
                            can_edit_old_attendance, is_admin
                        ) VALUES (?, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1)
                    """, (admin_id,))

    def setup_login_window(self):
        self.colors = {
            "primary": "#1E40AF",
            "secondary": "#3B82F6",
            "background": "#F1F5F9",
            "card": "#FFFFFF",
            "text": "#1F2937",
            "border": "#E5E7EB",
            "error": "#EF4444"
        }
        self.fonts = {
            "heading": ("Tajawal", 28, "bold"),
            "title": ("Tajawal", 18, "bold"),
            "normal": ("Tajawal", 14),
            "bold": ("Tajawal", 14, "bold"),
            "small": ("Tajawal", 12)
        }

        self.root.title(" نظام إدارة الدورات التخصصية - تسجيل الدخول")
        self.root.geometry("900x600")
        self.root.resizable(False, False)

        screen_width = self.root.winfo_screenwidth()
        screen_height = self.root.winfo_screenheight()
        x = (screen_width - 900) // 2
        y = (screen_height - 600) // 2
        self.root.geometry(f"900x600+{x}+{y}")

        main_frame = tk.Frame(self.root, bg=self.colors["background"])
        main_frame.pack(fill=tk.BOTH, expand=True)

        left_frame = tk.Frame(main_frame, bg=self.colors["primary"], width=350)
        left_frame.pack(side=tk.LEFT, fill=tk.Y)

        right_frame = tk.Frame(main_frame, bg=self.colors["background"])
        right_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

        left_title = tk.Label(
            left_frame,
            text="قــســم\nشــؤون\nالمـدربـين",
            font=self.fonts["heading"],
            bg=self.colors["primary"],
            fg="white",
            justify=tk.LEFT
        )
        left_title.place(x=30, y=150)

        left_footer = tk.Label(
            left_frame,
            text="© 2025\nجميع الحقوق محفوظة \n للمهندس / عبدالرحمن جفال الشمري ",
            font=self.fonts["small"],
            bg=self.colors["primary"],
            fg="white"
        )
        left_footer.place(x=30, y=520)

        card = tk.Frame(right_frame, bg=self.colors["card"], bd=1, relief=tk.RIDGE, padx=40, pady=30)
        card.place(relx=0.5, rely=0.5, anchor=tk.CENTER, width=420, height=380)

        login_label = tk.Label(card, text="تسجيل الدخول", font=self.fonts["title"], fg=self.colors["primary"],
                               bg=self.colors["card"])
        login_label.pack(pady=(0, 20))

        username_label = tk.Label(card, text="اسم المستخدم:", font=self.fonts["bold"], bg=self.colors["card"],
                                  fg=self.colors["text"])
        username_label.pack(anchor="w", pady=(5, 0))

        self.username_entry = tk.Entry(card, font=self.fonts["normal"], bg=self.colors["card"], fg=self.colors["text"],
                                       highlightthickness=1, highlightbackground=self.colors["border"], relief=tk.FLAT)
        self.username_entry.pack(fill=tk.X, pady=(0, 10), ipady=6)
        self.username_entry.focus_set()

        password_label = tk.Label(card, text="كلمة المرور:", font=self.fonts["bold"], bg=self.colors["card"],
                                  fg=self.colors["text"])
        password_label.pack(anchor="w", pady=(5, 0))

        self.password_entry = tk.Entry(card, font=self.fonts["normal"], bg=self.colors["card"], fg=self.colors["text"],
                                       highlightthickness=1, highlightbackground=self.colors["border"], show="•",
                                       relief=tk.FLAT)
        self.password_entry.pack(fill=tk.X, pady=(0, 20), ipady=6)
        self.password_entry.bind("<Return>", lambda event: self.login())

        login_button = tk.Button(card, text="دخول", font=self.fonts["bold"], bg=self.colors["secondary"], fg="white",
                                 bd=0, relief=tk.FLAT, cursor="hand2", command=self.login)
        login_button.pack(fill=tk.X, pady=(0, 10), ipady=8)

        self.status_label = tk.Label(card, text="", font=self.fonts["small"], bg=self.colors["card"],
                                     fg=self.colors["error"])
        self.status_label.pack()

    def login(self):
        username = self.username_entry.get().strip()
        password = self.password_entry.get().strip()

        if not username or not password:
            messagebox.showwarning("تنبيه", "الرجاء إدخال اسم المستخدم وكلمة المرور")
            return

        hashed_pwd = hashlib.sha256(password.encode()).hexdigest()
        cursor = self.db_conn.cursor()
        cursor.execute("""
            SELECT u.id, u.username, u.full_name
            FROM users u
            WHERE u.username=? AND u.password=? AND u.is_active=1
        """, (username, hashed_pwd))
        user = cursor.fetchone()

        if user:
            # قراءة الصلاحيات بالأسماء بدلاً من الأرقام
            cursor.execute("""
                SELECT 
                    user_id,
                    can_edit_attendance,
                    can_add_students,
                    can_edit_students,
                    can_delete_students,
                    can_view_edit_history,
                    can_reset_attendance,
                    can_export_data,
                    can_import_data,
                    is_admin,
                    can_edit_old_attendance
                FROM user_permissions 
                WHERE user_id=?
            """, (user[0],))

            perm_row = cursor.fetchone()

            if not perm_row:
                # إذا لم توجد صلاحيات، أنشئ صلاحيات افتراضية
                is_admin = 1 if username == 'admin' else 0
                with self.db_conn:
                    self.db_conn.execute("""
                        INSERT INTO user_permissions (
                            user_id, can_edit_attendance, can_add_students, 
                            can_edit_students, can_delete_students, can_view_edit_history,
                            can_reset_attendance, can_export_data, can_import_data, 
                            can_edit_old_attendance, is_admin
                        ) VALUES (?, 1, 1, 1, ?, ?, ?, 1, ?, ?, ?)
                    """, (user[0], is_admin, is_admin, is_admin, is_admin, 0, is_admin))

                # إعادة قراءة الصلاحيات
                cursor.execute("""
                    SELECT 
                        user_id,
                        can_edit_attendance,
                        can_add_students,
                        can_edit_students,
                        can_delete_students,
                        can_view_edit_history,
                        can_reset_attendance,
                        can_export_data,
                        can_import_data,
                        is_admin,
                        can_edit_old_attendance
                    FROM user_permissions 
                    WHERE user_id=?
                """, (user[0],))
                perm_row = cursor.fetchone()

            with self.db_conn:
                self.db_conn.execute("UPDATE users SET last_login=? WHERE id=?",
                                     (datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"), user[0]))

            # التأكد من أن المستخدم admin يحصل على صلاحيات المشرف
            if username == 'admin' and perm_row[9] != 1:  # is_admin
                with self.db_conn:
                    self.db_conn.execute("""
                        UPDATE user_permissions 
                        SET is_admin=1, can_edit_attendance=1, can_add_students=1,
                            can_edit_students=1, can_delete_students=1, can_view_edit_history=1,
                            can_reset_attendance=1, can_export_data=1, can_import_data=1,
                            can_edit_old_attendance=1
                        WHERE user_id=?
                    """, (user[0],))

                # إعادة قراءة الصلاحيات المحدثة
                cursor.execute("""
                    SELECT 
                        user_id,
                        can_edit_attendance,
                        can_add_students,
                        can_edit_students,
                        can_delete_students,
                        can_view_edit_history,
                        can_reset_attendance,
                        can_export_data,
                        can_import_data,
                        is_admin,
                        can_edit_old_attendance
                    FROM user_permissions 
                    WHERE user_id=?
                """, (user[0],))
                perm_row = cursor.fetchone()

            # بناء كائن المستخدم الحالي
            self.current_user = {
                "id": user[0],
                "username": user[1],
                "full_name": user[2],
                "permissions": {
                    "can_edit_attendance": bool(perm_row[1]),
                    "can_add_students": bool(perm_row[2]),
                    "can_edit_students": bool(perm_row[3]),
                    "can_delete_students": bool(perm_row[4]),
                    "can_view_edit_history": bool(perm_row[5]),
                    "can_reset_attendance": bool(perm_row[6]),
                    "can_export_data": bool(perm_row[7]),
                    "can_import_data": bool(perm_row[8]),
                    "is_admin": bool(perm_row[9]),
                    "can_edit_old_attendance": bool(perm_row[10])
                }
            }

            self.root.destroy()

            new_root = tk.Tk()
            ModernAttendanceSystem(new_root, self.current_user, self.db_conn)
            new_root.mainloop()
        else:
            messagebox.showwarning("خطأ", "اسم المستخدم أو كلمة المرور غير صحيحة")

    def verify_user_permissions(self, username):
        """دالة للتحقق من صلاحيات مستخدم معين"""
        cursor = self.db_conn.cursor()

        # الحصول على معرف المستخدم
        cursor.execute("SELECT id FROM users WHERE username=?", (username,))
        user = cursor.fetchone()

        if user:
            user_id = user[0]

            # قراءة الصلاحيات
            cursor.execute("""
                SELECT * FROM user_permissions WHERE user_id=?
            """, (user_id,))

            perms = cursor.fetchone()

            if perms:
                print(f"\n=== صلاحيات المستخدم {username} ===")
                cursor.execute("PRAGMA table_info(user_permissions)")
                columns = cursor.fetchall()

                for i, (cid, name, type_, notnull, dflt_value, pk) in enumerate(columns):
                    if i < len(perms):
                        print(f"{name}: {perms[i]}")
                print("================================\n")
            else:
                print(f"لا توجد صلاحيات محفوظة للمستخدم {username}")


class UserManagement:
    def __init__(self, root, conn, current_user, colors, fonts):
        self.root = root
        self.conn = conn
        self.current_user = current_user
        self.colors = colors
        self.fonts = fonts
        self.create_user_management_window()

    def create_user_management_window(self):
        self.user_window = tk.Toplevel(self.root)
        # تحديث وقت النشاط عند أي حركة في النافذة
        self.user_window.bind("<Motion>", lambda e: self.root.master.reset_activity_timer() if hasattr(self.root.master,
                                                                                                       'reset_activity_timer') else None)
        self.user_window.title("إدارة المستخدمين")
        self.user_window.geometry("900x700")
        self.user_window.configure(bg=self.colors["light"])
        # self.user_window.transient(self.root)  # قم بتعليق هذا السطر أو حذفه
        self.user_window.grab_set()

        # تفعيل خاصية تغيير حجم النافذة
        self.user_window.resizable(True, True)

        x = (self.user_window.winfo_screenwidth() - 900) // 2
        y = (self.user_window.winfo_screenheight() - 700) // 2
        self.user_window.geometry(f"900x700+{x}+{y}")

        tk.Label(
            self.user_window,
            text="إدارة المستخدمين",
            font=self.fonts["large_title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10, width=900
        ).pack(fill=tk.X)

        button_frame = tk.Frame(self.user_window, bg=self.colors["light"], pady=10)
        button_frame.pack(fill=tk.X, padx=10)

        add_user_btn = tk.Button(
            button_frame, text="إضافة مستخدم جديد", font=self.fonts["text_bold"], bg=self.colors["success"], fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.add_user
        )
        add_user_btn.pack(side=tk.RIGHT, padx=5)

        edit_user_btn = tk.Button(
            button_frame, text="تعديل المستخدم المحدد", font=self.fonts["text_bold"], bg=self.colors["warning"],
            fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.edit_user
        )
        edit_user_btn.pack(side=tk.RIGHT, padx=5)

        toggle_active_btn = tk.Button(
            button_frame, text="تفعيل/تعطيل المستخدم", font=self.fonts["text_bold"], bg=self.colors["secondary"],
            fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.toggle_user_active
        )
        toggle_active_btn.pack(side=tk.RIGHT, padx=5)

        delete_user_btn = tk.Button(
            button_frame, text="حذف المستخدم المحدد", font=self.fonts["text_bold"], bg=self.colors["danger"],
            fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.delete_user
        )
        delete_user_btn.pack(side=tk.RIGHT, padx=5)

        manage_permissions_btn = tk.Button(
            button_frame, text="إدارة صلاحيات المستخدم", font=self.fonts["text_bold"], bg="#9333EA", fg="white",
            padx=10, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.manage_user_permissions
        )
        manage_permissions_btn.pack(side=tk.RIGHT, padx=5)

        # إضافة زر الإشراف العام للأدمن فقط
        if self.current_user["username"] == "admin":
            supervision_btn = tk.Button(
                button_frame,
                text="إشراف عام",
                font=self.fonts["text_bold"],
                bg="#FF6B6B",  # لون أحمر مميز
                fg="white",
                padx=10, pady=5,
                bd=0, relief=tk.FLAT,
                cursor="hand2",
                command=self.open_general_supervision
            )
            supervision_btn.pack(side=tk.RIGHT, padx=5)

        table_frame = tk.Frame(self.user_window, bg=self.colors["light"], padx=10, pady=10)
        table_frame.pack(fill=tk.BOTH, expand=True)

        tree_scroll = tk.Scrollbar(table_frame)
        tree_scroll.pack(side=tk.RIGHT, fill=tk.Y)

        self.users_tree = ttk.Treeview(
            table_frame,
            columns=("id", "username", "full_name", "created_date", "last_login", "status", "is_admin"),
            show="headings",
            yscrollcommand=tree_scroll.set
        )
        self.users_tree.column("id", width=50, anchor=tk.CENTER)
        self.users_tree.column("username", width=120, anchor=tk.CENTER)
        self.users_tree.column("full_name", width=150, anchor=tk.CENTER)
        self.users_tree.column("created_date", width=120, anchor=tk.CENTER)
        self.users_tree.column("last_login", width=120, anchor=tk.CENTER)
        self.users_tree.column("status", width=80, anchor=tk.CENTER)
        self.users_tree.column("is_admin", width=80, anchor=tk.CENTER)

        self.users_tree.heading("id", text="الرقم")
        self.users_tree.heading("username", text="اسم المستخدم")
        self.users_tree.heading("full_name", text="الاسم الكامل")
        self.users_tree.heading("created_date", text="تاريخ الإنشاء")
        self.users_tree.heading("last_login", text="آخر تسجيل دخول")
        self.users_tree.heading("status", text="الحالة")
        self.users_tree.heading("is_admin", text="مشرف")

        self.users_tree.pack(fill=tk.BOTH, expand=True)
        tree_scroll.config(command=self.users_tree.yview)

        self.users_tree.tag_configure("active", background="#e8f5e9")
        self.users_tree.tag_configure("inactive", background="#ffebee")
        self.users_tree.tag_configure("admin", background="#e1f5fe")

        self.load_users()

        close_btn = tk.Button(
            self.user_window, text="إغلاق", font=self.fonts["text_bold"], bg=self.colors["dark"], fg="white",
            padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=self.user_window.destroy
        )
        close_btn.pack(pady=10)

    def open_general_supervision(self):
        """فتح نافذة الإشراف العام لمتابعة التعديلات التاريخية"""
        # طلب كلمة المرور
        password = simpledialog.askstring("كلمة مرور الإشراف العام",
                                          "أدخل كلمة مرور الإشراف العام:",
                                          show='*')
        if not password:
            return

        if password != "20255":
            messagebox.showerror("خطأ", "كلمة المرور غير صحيحة!")
            return

        # إنشاء نافذة الإشراف العام
        supervision_window = tk.Toplevel(self.user_window)
        supervision_window.title("الإشراف العام - متابعة التعديلات التاريخية")
        supervision_window.geometry("1200x700")
        supervision_window.configure(bg=self.colors["light"])
        supervision_window.transient(self.user_window)
        supervision_window.grab_set()

        # توسيط النافذة
        x = (supervision_window.winfo_screenwidth() - 1200) // 2
        y = (supervision_window.winfo_screenheight() - 700) // 2
        supervision_window.geometry(f"1200x700+{x}+{y}")

        # عنوان النافذة
        tk.Label(
            supervision_window,
            text="الإشراف العام - متابعة التعديلات على السجلات التاريخية",
            font=self.fonts["large_title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10
        ).pack(fill=tk.X)

        # إطار الفلترة
        filter_frame = tk.Frame(supervision_window, bg=self.colors["light"], padx=10, pady=10)
        filter_frame.pack(fill=tk.X)

        # فلتر التاريخ
        tk.Label(filter_frame, text="من تاريخ:", font=self.fonts["text_bold"],
                 bg=self.colors["light"]).pack(side=tk.RIGHT, padx=5)

        from_date = DateEntry(filter_frame, width=12, background=self.colors["primary"],
                              foreground='white', borderwidth=2, date_pattern='yyyy-mm-dd',
                              font=self.fonts["text"])
        from_date.pack(side=tk.RIGHT, padx=5)
        # تعيين تاريخ البداية قبل شهر
        from_date.set_date(datetime.datetime.now() - datetime.timedelta(days=30))

        tk.Label(filter_frame, text="إلى تاريخ:", font=self.fonts["text_bold"],
                 bg=self.colors["light"]).pack(side=tk.RIGHT, padx=5)

        to_date = DateEntry(filter_frame, width=12, background=self.colors["primary"],
                            foreground='white', borderwidth=2, date_pattern='yyyy-mm-dd',
                            font=self.fonts["text"])
        to_date.pack(side=tk.RIGHT, padx=5)

        # فلتر المستخدم
        tk.Label(filter_frame, text="المستخدم:", font=self.fonts["text_bold"],
                 bg=self.colors["light"]).pack(side=tk.RIGHT, padx=20)

        # الحصول على قائمة المستخدمين
        cursor = self.conn.cursor()
        cursor.execute("SELECT DISTINCT full_name FROM users WHERE username != 'admin' ORDER BY full_name")
        users = ["جميع المستخدمين"] + [row[0] for row in cursor.fetchall()]

        user_var = tk.StringVar(value="جميع المستخدمين")
        user_combo = ttk.Combobox(filter_frame, textvariable=user_var, values=users,
                                  state="readonly", width=25, font=self.fonts["text"])
        user_combo.pack(side=tk.RIGHT, padx=5)

        # زر البحث
        def search_edits():
            load_historical_edits()

        search_btn = tk.Button(filter_frame, text="بحث", font=self.fonts["text_bold"],
                               bg=self.colors["success"], fg="white", padx=15, pady=5,
                               bd=0, relief=tk.FLAT, cursor="hand2", command=search_edits)
        search_btn.pack(side=tk.LEFT, padx=10)

        # زر التصدير
        export_btn = tk.Button(filter_frame, text="تصدير Excel", font=self.fonts["text_bold"],
                               bg=self.colors["secondary"], fg="white", padx=15, pady=5,
                               bd=0, relief=tk.FLAT, cursor="hand2",
                               command=lambda: export_historical_edits())
        export_btn.pack(side=tk.LEFT, padx=5)

        # إطار الجدول
        table_frame = tk.Frame(supervision_window, bg=self.colors["light"], padx=10, pady=10)
        table_frame.pack(fill=tk.BOTH, expand=True)

        # شريط التمرير
        tree_scroll = tk.Scrollbar(table_frame)
        tree_scroll.pack(side=tk.RIGHT, fill=tk.Y)

        # إنشاء الجدول
        edits_tree = ttk.Treeview(
            table_frame,
            columns=("id", "edit_date", "edit_time", "editor", "student", "national_id",
                     "original_date", "old_status", "new_status", "days_diff"),
            show="headings",
            yscrollcommand=tree_scroll.set,
            style="Bold.Treeview"
        )

        # تعريف الأعمدة
        edits_tree.column("id", width=50, anchor=tk.CENTER)
        edits_tree.column("edit_date", width=100, anchor=tk.CENTER)
        edits_tree.column("edit_time", width=80, anchor=tk.CENTER)
        edits_tree.column("editor", width=150, anchor=tk.CENTER)
        edits_tree.column("student", width=150, anchor=tk.CENTER)
        edits_tree.column("national_id", width=100, anchor=tk.CENTER)
        edits_tree.column("original_date", width=100, anchor=tk.CENTER)
        edits_tree.column("old_status", width=80, anchor=tk.CENTER)
        edits_tree.column("new_status", width=80, anchor=tk.CENTER)
        edits_tree.column("days_diff", width=80, anchor=tk.CENTER)

        # عناوين الأعمدة
        edits_tree.heading("id", text="#")
        edits_tree.heading("edit_date", text="تاريخ التعديل")
        edits_tree.heading("edit_time", text="وقت التعديل")
        edits_tree.heading("editor", text="من قام بالتعديل")
        edits_tree.heading("student", text="اسم المتدرب")
        edits_tree.heading("national_id", text="رقم الهوية")
        edits_tree.heading("original_date", text="التاريخ الأصلي")
        edits_tree.heading("old_status", text="الحالة القديمة")
        edits_tree.heading("new_status", text="الحالة الجديدة")
        edits_tree.heading("days_diff", text="عدد الأيام")

        edits_tree.pack(fill=tk.BOTH, expand=True)
        tree_scroll.config(command=edits_tree.yview)

        # تطبيق ألوان مختلفة حسب نوع التعديل
        edits_tree.tag_configure("critical", background="#ffebee")  # تعديلات قديمة جداً
        edits_tree.tag_configure("warning", background="#fff8e1")  # تعديلات متوسطة
        edits_tree.tag_configure("normal", background="#e8f5e9")  # تعديلات حديثة

        # دالة تحميل البيانات
        def load_historical_edits():
            # مسح البيانات الحالية
            for item in edits_tree.get_children():
                edits_tree.delete(item)

            # بناء الاستعلام
            query = """
                SELECT id, attendance_id, national_id, student_name, edit_date, 
                       original_date, old_status, new_status, edited_by, 
                       edit_timestamp, days_difference
                FROM historical_edits_log
                WHERE edit_date BETWEEN ? AND ?
            """
            params = [from_date.get_date().strftime("%Y-%m-%d"),
                      to_date.get_date().strftime("%Y-%m-%d")]

            # إضافة فلتر المستخدم
            if user_var.get() != "جميع المستخدمين":
                query += " AND edited_by = ?"
                params.append(user_var.get())

            query += " ORDER BY edit_timestamp DESC"

            cursor = self.conn.cursor()
            cursor.execute(query, params)
            records = cursor.fetchall()

            # عرض البيانات
            for i, record in enumerate(records):
                # التعامل مع الفهارس بشكل صحيح
                rec_id = record[0]
                attendance_id = record[1]
                national_id = record[2]
                student_name = record[3]
                edit_date_str = record[4]
                original_date = record[5]
                old_status = record[6]
                new_status = record[7]
                edited_by = record[8]
                edit_timestamp = record[9]
                days_difference = record[10]

                # تنسيق التاريخ
                try:
                    edit_date = datetime.datetime.strptime(edit_date_str, "%Y-%m-%d").strftime("%Y/%m/%d")
                except:
                    edit_date = edit_date_str

                # تنسيق الوقت
                try:
                    edit_time = edit_timestamp.split()[1] if len(edit_timestamp.split()) > 1 else edit_timestamp
                except:
                    edit_time = "غير محدد"

                values = (
                    i + 1,
                    edit_date,
                    edit_time,
                    edited_by,
                    student_name,
                    national_id,
                    original_date,
                    old_status,
                    new_status,
                    f"{days_difference} يوم"
                )

                item = edits_tree.insert("", tk.END, values=values)

                # تطبيق اللون حسب عدد الأيام
                if days_difference > 7:  # أكثر من أسبوع
                    edits_tree.item(item, tags=("critical",))
                elif days_difference > 3:  # أكثر من 3 أيام
                    edits_tree.item(item, tags=("warning",))
                else:
                    edits_tree.item(item, tags=("normal",))

            # إظهار عدد السجلات
            count_label.config(text=f"عدد التعديلات: {len(records)}")

        # دالة التصدير
        def export_historical_edits():
            # بناء الاستعلام نفسه
            query = """
                SELECT id, attendance_id, national_id, student_name, edit_date, 
                       original_date, old_status, new_status, edited_by, 
                       edit_timestamp, days_difference
                FROM historical_edits_log
                WHERE edit_date BETWEEN ? AND ?
            """
            params = [from_date.get_date().strftime("%Y-%m-%d"),
                      to_date.get_date().strftime("%Y-%m-%d")]

            if user_var.get() != "جميع المستخدمين":
                query += " AND edited_by = ?"
                params.append(user_var.get())

            query += " ORDER BY edit_timestamp DESC"

            # قراءة البيانات
            df = pd.read_sql(query, self.conn, params=params)

            if df.empty:
                messagebox.showinfo("تنبيه", "لا توجد بيانات للتصدير")
                return

            # تنسيق الأعمدة
            df = df.rename(columns={
                'id': 'م',
                'attendance_id': 'معرف السجل',
                'national_id': 'رقم الهوية',
                'student_name': 'اسم المتدرب',
                'edit_date': 'تاريخ التعديل',
                'original_date': 'التاريخ الأصلي',
                'old_status': 'الحالة القديمة',
                'new_status': 'الحالة الجديدة',
                'edited_by': 'المستخدم',
                'edit_timestamp': 'وقت التعديل الكامل',
                'days_difference': 'عدد الأيام'
            })

            # اختيار مسار الحفظ
            export_file = filedialog.asksaveasfilename(
                defaultextension=".xlsx",
                filetypes=[("Excel files", "*.xlsx")],
                initialfile=f"سجل_التعديلات_التاريخية_{datetime.datetime.now().strftime('%Y%m%d')}.xlsx"
            )

            if export_file:
                try:
                    df.to_excel(export_file, index=False)
                    messagebox.showinfo("نجاح", f"تم تصدير البيانات إلى:\n{export_file}")
                except Exception as e:
                    messagebox.showerror("خطأ", f"حدث خطأ أثناء التصدير: {str(e)}")

        # عداد السجلات
        count_label = tk.Label(supervision_window, text="عدد التعديلات: 0",
                               font=self.fonts["text_bold"], bg=self.colors["light"])
        count_label.pack(pady=5)

        # زر الإغلاق
        close_btn = tk.Button(supervision_window, text="إغلاق", font=self.fonts["text_bold"],
                              bg=self.colors["dark"], fg="white", padx=20, pady=5,
                              bd=0, relief=tk.FLAT, cursor="hand2",
                              command=supervision_window.destroy)
        close_btn.pack(pady=10)

        # تحميل البيانات عند فتح النافذة
        load_historical_edits()

    def load_users(self):
        for item in self.users_tree.get_children():
            self.users_tree.delete(item)
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT u.id, u.username, u.full_name, u.created_date, u.last_login, u.is_active,
                   COALESCE(p.is_admin, 0) as is_admin
            FROM users u
            LEFT JOIN user_permissions p ON u.id = p.user_id
        """)
        users = cursor.fetchall()
        for user in users:
            user_id, username, full_name, created_date, last_login, is_active, is_admin = user
            status = "نشط" if is_active else "معطل"
            admin_status = "نعم" if is_admin else "لا"
            if not last_login:
                last_login = "لم يسجل الدخول بعد"
            item_id = self.users_tree.insert("", tk.END, values=(
                user_id, username, full_name, created_date, last_login, status, admin_status))

            if not is_active:
                self.users_tree.item(item_id, tags=("inactive",))
            elif is_admin:
                self.users_tree.item(item_id, tags=("admin",))
            else:
                self.users_tree.item(item_id, tags=("active",))

    def add_user(self):
        add_window = tk.Toplevel(self.user_window)
        add_window.bind("<Motion>", lambda e: self.root.master.reset_activity_timer() if hasattr(self.root.master,
                                                                                                 'reset_activity_timer') else None)
        add_window.title("إضافة مستخدم جديد")
        add_window.geometry("400x430")
        add_window.configure(bg=self.colors["light"])
        add_window.transient(self.user_window)
        add_window.grab_set()

        x = (add_window.winfo_screenwidth() - 400) // 2
        y = (add_window.winfo_screenheight() - 430) // 2
        add_window.geometry(f"400x430+{x}+{y}")

        tk.Label(
            add_window,
            text="إضافة مستخدم جديد",
            font=self.fonts["title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10, width=400
        ).pack(fill=tk.X)

        form_frame = tk.Frame(add_window, bg=self.colors["light"], padx=20, pady=20)
        form_frame.pack(fill=tk.BOTH)

        tk.Label(form_frame, text="اسم المستخدم:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=0, column=1, padx=5, pady=8, sticky=tk.E)
        username_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        username_entry.grid(row=0, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="الاسم الكامل:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=1, column=1, padx=5, pady=8, sticky=tk.E)
        fullname_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        fullname_entry.grid(row=1, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="كلمة المرور:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=2, column=1, padx=5, pady=8, sticky=tk.E)
        password_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25, show="*")
        password_entry.grid(row=2, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="تأكيد كلمة المرور:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=3, column=1, padx=5, pady=8, sticky=tk.E)
        confirm_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25, show="*")
        confirm_entry.grid(row=3, column=0, padx=5, pady=8, sticky=tk.W)

        is_admin_var = tk.IntVar(value=0)
        admin_check = tk.Checkbutton(
            form_frame,
            text="جعل هذا المستخدم مشرفًا",
            variable=is_admin_var,
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        admin_check.grid(row=4, column=0, columnspan=2, padx=5, pady=8, sticky=tk.W)

        button_frame = tk.Frame(add_window, bg=self.colors["light"], pady=10)
        button_frame.pack(fill=tk.X)

        def save_user():
            username = username_entry.get().strip()
            fullname = fullname_entry.get().strip()
            password = password_entry.get().strip()
            confirm = confirm_entry.get().strip()
            is_admin = is_admin_var.get()

            if not all([username, fullname, password, confirm]):
                messagebox.showwarning("تنبيه", "يجب ملء جميع الحقول")
                return
            if password != confirm:
                messagebox.showwarning("تنبيه", "كلمات المرور غير متطابقة")
                return
            if not re.match(r'^[a-zA-Z0-9_-]+$', username):
                messagebox.showwarning("تنبيه", "اسم المستخدم يجب أن يتكون من حروف إنجليزية وأرقام وشرطات فقط")
                return
            cursor = self.conn.cursor()
            cursor.execute("SELECT COUNT(*) FROM users WHERE username=?", (username,))
            count = cursor.fetchone()[0]
            if count > 0:
                messagebox.showwarning("تنبيه", "اسم المستخدم موجود بالفعل")
                return

            hashed_pwd = hashlib.sha256(password.encode()).hexdigest()
            try:
                with self.conn:
                    self.conn.execute("""
                        INSERT INTO users (username, password, full_name, created_date, is_active)
                        VALUES (?, ?, ?, ?, ?)
                    """, (
                        username,
                        hashed_pwd,
                        fullname,
                        datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                        1
                    ))

                    cursor.execute("SELECT id FROM users WHERE username=?", (username,))
                    user_id = cursor.fetchone()[0]

                    if is_admin:
                        self.conn.execute("""
                            INSERT INTO user_permissions (
                                user_id, can_edit_attendance, can_add_students, 
                                can_edit_students, can_delete_students, can_view_edit_history,
                                can_reset_attendance, can_export_data, can_import_data, is_admin
                            ) VALUES (?, 1, 1, 1, 1, 1, 1, 1, 1, 1)
                        """, (user_id,))
                    else:
                        self.conn.execute("""
                            INSERT INTO user_permissions (
                                user_id, can_edit_attendance, can_add_students, 
                                can_edit_students, can_delete_students, can_view_edit_history,
                                can_reset_attendance, can_export_data, can_import_data, is_admin
                            ) VALUES (?, 1, 1, 1, 0, 0, 0, 1, 0, 0)
                        """, (user_id,))

                messagebox.showinfo("نجاح", "تم إضافة المستخدم بنجاح")
                add_window.destroy()
                self.load_users()
            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء إضافة المستخدم: {str(e)}")

        save_btn = tk.Button(button_frame, text="حفظ", font=self.fonts["text_bold"], bg=self.colors["success"],
                             fg="white",
                             padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=save_user)
        save_btn.pack(side=tk.LEFT, padx=10)
        cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                               fg="white",
                               padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=add_window.destroy)
        cancel_btn.pack(side=tk.RIGHT, padx=10)

    def edit_user(self):
        selected_item = self.users_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد مستخدم من القائمة")
            return
        values = self.users_tree.item(selected_item, "values")
        user_id = values[0]
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id=?", (user_id,))
        user = cursor.fetchone()
        if not user:
            messagebox.showerror("خطأ", "لم يتم العثور على المستخدم")
            return

        cursor.execute("SELECT * FROM user_permissions WHERE user_id=?", (user_id,))
        permissions = cursor.fetchone()
        is_admin = 0
        if permissions:
            is_admin = permissions[9]

        edit_window = tk.Toplevel(self.user_window)
        edit_window.bind("<Motion>", lambda e: self.root.master.reset_activity_timer() if hasattr(self.root.master,
                                                                                                  'reset_activity_timer') else None)
        edit_window.title("تعديل المستخدم")
        edit_window.geometry("400x430")
        edit_window.configure(bg=self.colors["light"])
        edit_window.transient(self.user_window)
        edit_window.grab_set()

        x = (edit_window.winfo_screenwidth() - 400) // 2
        y = (edit_window.winfo_screenheight() - 430) // 2
        edit_window.geometry(f"400x430+{x}+{y}")

        tk.Label(
            edit_window,
            text=f"تعديل المستخدم: {user[1]}",
            font=self.fonts["title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10, width=400
        ).pack(fill=tk.X)

        form_frame = tk.Frame(edit_window, bg=self.colors["light"], padx=20, pady=20)
        form_frame.pack(fill=tk.BOTH)

        tk.Label(form_frame, text="اسم المستخدم:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=0, column=1, padx=5, pady=8, sticky=tk.E)
        username_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        username_entry.insert(0, user[1])
        username_entry.grid(row=0, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="الاسم الكامل:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=1, column=1, padx=5, pady=8, sticky=tk.E)
        fullname_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25)
        fullname_entry.insert(0, user[3])
        fullname_entry.grid(row=1, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="كلمة المرور الجديدة:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=2, column=1, padx=5, pady=8, sticky=tk.E)
        password_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25, show="*")
        password_entry.grid(row=2, column=0, padx=5, pady=8, sticky=tk.W)

        tk.Label(form_frame, text="تأكيد كلمة المرور:", font=self.fonts["text_bold"], bg=self.colors["light"],
                 anchor=tk.E).grid(row=3, column=1, padx=5, pady=8, sticky=tk.E)
        confirm_entry = tk.Entry(form_frame, font=self.fonts["text"], width=25, show="*")
        confirm_entry.grid(row=3, column=0, padx=5, pady=8, sticky=tk.W)

        is_admin_var = tk.IntVar(value=is_admin)
        admin_check = tk.Checkbutton(
            form_frame,
            text="هذا المستخدم مشرف",
            variable=is_admin_var,
            font=self.fonts["text"],
            bg=self.colors["light"]
        )
        admin_check.grid(row=4, column=0, columnspan=2, padx=5, pady=8, sticky=tk.W)

        button_frame = tk.Frame(edit_window, bg=self.colors["light"], pady=10)
        button_frame.pack(fill=tk.X)

        def save_changes():
            username = username_entry.get().strip()
            fullname = fullname_entry.get().strip()
            password = password_entry.get().strip()
            confirm = confirm_entry.get().strip()
            is_admin = is_admin_var.get()

            if not all([username, fullname]):
                messagebox.showwarning("تنبيه", "يجب ملء الحقول الأساسية")
                return

            if password:
                if password != confirm:
                    messagebox.showwarning("تنبيه", "كلمات المرور غير متطابقة")
                    return
            try:
                with self.conn:
                    if password:
                        hashed_pwd = hashlib.sha256(password.encode()).hexdigest()
                        self.conn.execute("UPDATE users SET username=?, full_name=?, password=? WHERE id=?",
                                          (username, fullname, hashed_pwd, user[0]))
                    else:
                        self.conn.execute("UPDATE users SET username=?, full_name=? WHERE id=?",
                                          (username, fullname, user[0]))

                    cursor = self.conn.cursor()
                    cursor.execute("SELECT COUNT(*) FROM user_permissions WHERE user_id=?", (user[0],))
                    has_permissions = cursor.fetchone()[0] > 0

                    if has_permissions:
                        self.conn.execute("UPDATE user_permissions SET is_admin=? WHERE user_id=?",
                                          (is_admin, user[0]))
                    else:
                        if is_admin:
                            self.conn.execute("""
                                INSERT INTO user_permissions (
                                    user_id, can_edit_attendance, can_add_students, 
                                    can_edit_students, can_delete_students, can_view_edit_history,
                                    can_reset_attendance, can_export_data, can_import_data, is_admin
                                ) VALUES (?, 1, 1, 1, 1, 1, 1, 1, 1, 1)
                            """, (user[0],))
                        else:
                            self.conn.execute("""
                                INSERT INTO user_permissions (
                                    user_id, can_edit_attendance, can_add_students, 
                                    can_edit_students, can_delete_students, can_view_edit_history,
                                    can_reset_attendance, can_export_data, can_import_data, is_admin
                                ) VALUES (?, 1, 1, 1, 0, 0, 0, 1, 0, 0)
                            """, (user[0],))

                messagebox.showinfo("نجاح", "تم تحديث بيانات المستخدم بنجاح")
                edit_window.destroy()
                self.load_users()
            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء تحديث المستخدم: {str(e)}")

        save_btn = tk.Button(button_frame, text="حفظ التغييرات", font=self.fonts["text_bold"],
                             bg=self.colors["warning"], fg="white",
                             padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=save_changes)
        save_btn.pack(side=tk.LEFT, padx=10)
        cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                               fg="white",
                               padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2", command=edit_window.destroy)
        cancel_btn.pack(side=tk.RIGHT, padx=10)

    def toggle_user_active(self):
        selected_item = self.users_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد مستخدم من القائمة")
            return
        values = self.users_tree.item(selected_item, "values")
        user_id = values[0]
        username = values[1]
        status_text = values[5]
        if username == self.current_user["username"]:
            messagebox.showwarning("تنبيه", "لا يمكن تعطيل المستخدم الحالي")
            return
        new_status = 0 if status_text == "نشط" else 1
        status_msg = "تفعيل" if new_status == 1 else "تعطيل"
        if not messagebox.askyesnocancel("تأكيد", f"هل تريد {status_msg} المستخدم {username}؟"):
            return
        try:
            with self.conn:
                self.conn.execute("UPDATE users SET is_active=? WHERE id=?", (new_status, user_id))
            messagebox.showinfo("نجاح", f"تم {status_msg} المستخدم بنجاح")
            self.load_users()
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ: {str(e)}")

    def delete_user(self):
        selected_item = self.users_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد مستخدم من القائمة")
            return
        values = self.users_tree.item(selected_item, "values")
        user_id = values[0]
        username = values[1]
        if username == self.current_user["username"]:
            messagebox.showwarning("تنبيه", "لا يمكن حذف المستخدم الحالي")
            return
        if not messagebox.askyesnocancel("تأكيد", f"هل تريد حذف المستخدم {username}؟\nلا يمكن التراجع عن العملية!"):
            return
        try:
            with self.conn:
                self.conn.execute("DELETE FROM user_permissions WHERE user_id=?", (user_id,))
                self.conn.execute("DELETE FROM users WHERE id=?", (user_id,))
            messagebox.showinfo("نجاح", "تم حذف المستخدم بنجاح")
            self.load_users()
        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء حذف المستخدم: {str(e)}")

    def manage_user_permissions(self):
        selected_item = self.users_tree.selection()
        if not selected_item:
            messagebox.showinfo("تنبيه", "الرجاء تحديد مستخدم من القائمة")
            return
        values = self.users_tree.item(selected_item, "values")
        user_id = values[0]
        username = values[1]

        cursor = self.conn.cursor()

        # قراءة الصلاحيات بالأسماء بدلاً من SELECT *
        cursor.execute("""
            SELECT 
                can_edit_attendance,
                can_add_students,
                can_edit_students,
                can_delete_students,
                can_view_edit_history,
                can_reset_attendance,
                can_export_data,
                can_import_data,
                is_admin,
                can_edit_old_attendance
            FROM user_permissions 
            WHERE user_id=?
        """, (user_id,))

        permissions = cursor.fetchone()

        if not permissions:
            # إنشاء صلاحيات افتراضية إذا لم تكن موجودة
            is_admin = 1 if values[6] == "نعم" else 0
            with self.conn:
                cursor.execute("""
                    INSERT INTO user_permissions (
                        user_id, can_edit_attendance, can_add_students, 
                        can_edit_students, can_delete_students, can_view_edit_history,
                        can_reset_attendance, can_export_data, can_import_data, is_admin,
                        can_edit_old_attendance
                    ) VALUES (?, 1, 1, 1, ?, ?, ?, 1, ?, ?, ?)
                """, (user_id, is_admin, is_admin, is_admin, is_admin, is_admin, 0))

            cursor.execute("""
                SELECT 
                    can_edit_attendance,
                    can_add_students,
                    can_edit_students,
                    can_delete_students,
                    can_view_edit_history,
                    can_reset_attendance,
                    can_export_data,
                    can_import_data,
                    is_admin,
                    can_edit_old_attendance
                FROM user_permissions 
                WHERE user_id=?
            """, (user_id,))
            permissions = cursor.fetchone()

        perm_window = tk.Toplevel(self.user_window)
        perm_window.bind("<Motion>", lambda e: self.root.master.reset_activity_timer() if hasattr(self.root.master,
                                                                                                  'reset_activity_timer') else None)
        perm_window.title(f"إدارة صلاحيات المستخدم: {username}")
        perm_window.geometry("500x600")
        perm_window.configure(bg=self.colors["light"])
        perm_window.transient(self.user_window)
        perm_window.grab_set()

        x = (perm_window.winfo_screenwidth() - 500) // 2
        y = (perm_window.winfo_screenheight() - 600) // 2
        perm_window.geometry(f"500x600+{x}+{y}")

        tk.Label(
            perm_window,
            text=f"صلاحيات المستخدم: {username}",
            font=self.fonts["title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10
        ).pack(fill=tk.X)

        perm_frame = tk.Frame(perm_window, bg=self.colors["light"], padx=20, pady=20)
        perm_frame.pack(fill=tk.BOTH, expand=True)

        # متغيرات الصلاحيات - القراءة بالترتيب الصحيح
        is_admin_var = tk.IntVar(value=permissions[8])  # is_admin
        can_edit_attendance_var = tk.IntVar(value=permissions[0])
        can_add_students_var = tk.IntVar(value=permissions[1])
        can_edit_students_var = tk.IntVar(value=permissions[2])
        can_delete_students_var = tk.IntVar(value=permissions[3])
        can_view_edit_history_var = tk.IntVar(value=permissions[4])
        can_reset_attendance_var = tk.IntVar(value=permissions[5])
        can_export_data_var = tk.IntVar(value=permissions[6])
        can_import_data_var = tk.IntVar(value=permissions[7])
        can_edit_old_attendance_var = tk.IntVar(value=permissions[9])  # can_edit_old_attendance

        def update_permissions():
            is_admin = is_admin_var.get()
            if is_admin:
                for var in [can_edit_attendance_var, can_add_students_var, can_edit_students_var,
                            can_delete_students_var, can_view_edit_history_var, can_reset_attendance_var,
                            can_export_data_var, can_import_data_var, can_edit_old_attendance_var]:
                    var.set(1)

                for checkbox in permission_checkboxes:
                    checkbox.config(state=tk.DISABLED)
            else:
                for checkbox in permission_checkboxes:
                    checkbox.config(state=tk.NORMAL)

        admin_title = tk.Label(perm_frame, text="صلاحيات عامة:", font=self.fonts["text_bold"], bg=self.colors["light"])
        admin_title.grid(row=0, column=0, sticky=tk.W, pady=(0, 10))

        admin_check = tk.Checkbutton(
            perm_frame,
            text="هذا المستخدم مشرف (يملك كل الصلاحيات)",
            variable=is_admin_var,
            font=self.fonts["text_bold"],
            bg=self.colors["light"],
            command=update_permissions
        )
        admin_check.grid(row=1, column=0, sticky=tk.W, pady=5)

        specific_title = tk.Label(perm_frame, text="صلاحيات محددة:", font=self.fonts["text_bold"],
                                  bg=self.colors["light"])
        specific_title.grid(row=2, column=0, sticky=tk.W, pady=(20, 10))

        permission_options = [
            (can_edit_attendance_var, "تعديل سجلات الحضور والغياب"),
            (can_add_students_var, "إضافة متدربين جدد"),
            (can_edit_students_var, "تعديل بيانات المتدربين"),
            (can_delete_students_var, "حذف المتدربين"),
            (can_view_edit_history_var, "عرض سجل التعديلات (من عدّل ومتى)"),
            (can_reset_attendance_var, "إعادة تعيين سجلات الحضور"),
            (can_export_data_var, "تصدير البيانات"),
            (can_import_data_var, "استيراد البيانات من Excel"),
            (can_edit_old_attendance_var, "تعديل سجلات الحضور القديمة (أكثر من يوم)")
        ]

        permission_checkboxes = []
        for i, (var, text) in enumerate(permission_options):
            checkbox = tk.Checkbutton(
                perm_frame,
                text=text,
                variable=var,
                font=self.fonts["text"],
                bg=self.colors["light"]
            )
            checkbox.grid(row=i + 3, column=0, sticky=tk.W, pady=5)
            permission_checkboxes.append(checkbox)

        update_permissions()

        button_frame = tk.Frame(perm_window, bg=self.colors["light"], pady=10)
        button_frame.pack(fill=tk.X, padx=20)

        def save_permissions():
            try:
                with self.conn:
                    self.conn.execute("""
                        UPDATE user_permissions SET
                            is_admin=?,
                            can_edit_attendance=?,
                            can_add_students=?,
                            can_edit_students=?,
                            can_delete_students=?,
                            can_view_edit_history=?,
                            can_reset_attendance=?,
                            can_export_data=?,
                            can_import_data=?,
                            can_edit_old_attendance=?
                        WHERE user_id=?
                    """, (
                        is_admin_var.get(),
                        can_edit_attendance_var.get(),
                        can_add_students_var.get(),
                        can_edit_students_var.get(),
                        can_delete_students_var.get(),
                        can_view_edit_history_var.get(),
                        can_reset_attendance_var.get(),
                        can_export_data_var.get(),
                        can_import_data_var.get(),
                        can_edit_old_attendance_var.get(),
                        user_id
                    ))
                messagebox.showinfo("نجاح", "تم تحديث صلاحيات المستخدم بنجاح")
                perm_window.destroy()
                self.load_users()
            except Exception as e:
                messagebox.showerror("خطأ", f"حدث خطأ أثناء تحديث الصلاحيات: {str(e)}")

        save_btn = tk.Button(button_frame, text="حفظ الصلاحيات", font=self.fonts["text_bold"],
                             bg=self.colors["success"],
                             fg="white", padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2",
                             command=save_permissions)
        save_btn.pack(side=tk.LEFT, padx=10)

        cancel_btn = tk.Button(button_frame, text="إلغاء", font=self.fonts["text_bold"], bg=self.colors["danger"],
                               fg="white", padx=15, pady=5, bd=0, relief=tk.FLAT, cursor="hand2",
                               command=perm_window.destroy)
        cancel_btn.pack(side=tk.RIGHT, padx=10)


class ModernAttendanceSystem:
    def __init__(self, root, current_user, conn=None):
        self.root = root
        self.current_user = current_user
        self.root.title("نظام إدارة الدورات التخصصية")

        # تخزين الحجم الأصلي للخطوط قبل التعديل
        self.original_fonts = {
            "large_title": ("Tajawal", 24, "bold"),
            "title": ("Tajawal", 18, "bold"),
            "subtitle": ("Tajawal", 16, "bold"),
            "text": ("Tajawal", 12),
            "text_bold": ("Tajawal", 12, "bold"),
            "small": ("Tajawal", 10)
        }

        # تعريف الألوان
        self.colors = {
            "primary": "#1a73e8",
            "secondary": "#4285f4",
            "success": "#34a853",
            "danger": "#ea4335",
            "warning": "#fbbc05",
            "light": "#f0f4f8",
            "dark": "#202124",
            "present": "#34a853",
            "absent": "#ea4335",
            "late": "#fbbc05",
            "excused": "#4285f4",
            "not_started": "#FFA500",
            "excluded": "#9C27B0",
            "field_application": "#909090",
            "student_day": "#A9A9A9",
            "evening_remote": "#A0A0A0",
            "death_case": "#7E57C2",
            "hospital": "#26A69A",
        }

        # تحديد التخطيط الأمثل بناءً على حجم الشاشة
        self.determine_best_layout()

        # تعريف الخطوط بعد تحديد الحجم المناسب
        self.fonts = self.original_fonts.copy()

        self.style = ttk.Style(self.root)
        self.style.theme_use("clam")
        self.setup_styles()

        # ربط حدث تغيير حجم النافذة بدالة التكيف التلقائي
        self.root.bind('<Configure>', self.on_window_resize)

        self.tab_control = ttk.Notebook(self.root, style="Bold.TNotebook")
        self.tab_control.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

        if conn:
            self.conn = conn
        else:
            self.conn = sqlite3.connect("attendance.db")

        self.create_tables()
        self.create_indexes()

        self.today = datetime.datetime.now().strftime("%Y-%m-%d")

        # تعريف متغيرات الإحصائيات
        self.total_students_var = tk.StringVar(value="0")
        self.present_students_var = tk.StringVar(value="0")
        self.absent_students_var = tk.StringVar(value="0")
        self.late_students_var = tk.StringVar(value="0")
        self.excused_students_var = tk.StringVar(value="0")
        self.not_started_students_var = tk.StringVar(value="0")
        self.field_application_var = tk.StringVar(value="0")
        self.student_day_var = tk.StringVar(value="0")
        self.evening_remote_var = tk.StringVar(value="0")
        self.attendance_rate_var = tk.StringVar(value="0%")
        self.death_case_var = tk.StringVar(value="0")
        self.hospital_var = tk.StringVar(value="0")

        # تخزين إشارات لبطاقات الإحصائيات للتحكم فيها لاحقًا
        self.stats_cards = []

        self.create_header()

        self.attendance_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
        self.tab_control.add(self.attendance_tab, text="سجل الحضور")

        self.attendance_log_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
        self.tab_control.add(self.attendance_log_tab, text="استعراض الحضور")

        self.students_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
        self.tab_control.add(self.students_tab, text="إدارة المتدربين")

        # إعداد قاعدة البيانات
        if conn:
            self.conn = conn
        else:
            # فتح الاتصال باستخدام خيارات تحسين الأداء
            self.conn = sqlite3.connect("attendance.db", isolation_level=None)

            # تحسين أداء قاعدة البيانات
            self.conn.execute("PRAGMA journal_mode = WAL")  # استخدام وضع WAL للتخزين
            self.conn.execute("PRAGMA synchronous = NORMAL")  # تقليل وقت الانتظار للكتابة
            self.conn.execute("PRAGMA cache_size = -20000")  # استخدام ذاكرة تخزين مؤقت أكبر (حوالي 20 ميجابايت)
            self.conn.execute("PRAGMA temp_store = MEMORY")  # استخدام الذاكرة للتخزين المؤقت

        # إنشاء وتحسين الجداول والفهارس
        self.create_tables()
        self.create_indexes()  # دالة جديدة تمت إضافتها

        if self.current_user["permissions"]["is_admin"]:
            self.users_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
            self.tab_control.add(self.users_tab, text="إدارة المستخدمين")
            self.setup_users_tab()

        self.setup_attendance_tab()
        self.setup_attendance_log_tab()
        self.setup_students_tab()

        self.status_bar = tk.Label(
            self.root,
            text=f"مرحبًا {self.current_user['full_name']} (مستخدم: {self.current_user['username']})",
            font=self.fonts["small"], bg=self.colors["primary"], fg="white", pady=5
        )
        self.status_bar.pack(side=tk.BOTTOM, fill=tk.X)

        self.archive_manager = ArchiveManager(self.root, self, self.colors, self.fonts)

        # في class AttendanceApp, في دالة __init__
        # ابحث عن هذا الجزء:

        # إضافة تبويب الأرشيف
        self.archive_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
        self.tab_control.add(self.archive_tab, text="أرشيف الدورات")
        self.setup_archive_tab()

        # إضافة تبويب مراقبة الغياب
        add_absence_monitoring_icon(self)

        # إضافة متغيرات تتبع النشاط لتسجيل الخروج التلقائي - الجزء الجديد
        self.last_activity_time = time.time()

        # إضافة متغيرات تتبع النشاط لتسجيل الخروج التلقائي - الجزء الجديد
        self.last_activity_time = time.time()
        self.inactivity_timeout = 600  # 30 ثانية للتجربة (يمكن تغييرها إلى 1200 للإعداد النهائي - 20 دقيقة)
        self.activity_check_id = None

        # ربط حركات المستخدم بتحديث وقت النشاط - الجزء الجديد
        self.root.bind("<Motion>", self.reset_activity_timer)
        self.root.bind("<Button-1>", self.reset_activity_timer)
        self.root.bind("<ButtonRelease-1>", self.reset_activity_timer)
        self.root.bind("<Key>", self.reset_activity_timer)

        # ربط دالة إغلاق النافذة - الجزء الجديد
        self.root.protocol("WM_DELETE_WINDOW", self.on_closing)

        # إضافة متغيرات لتتبع آخر عمليات التسجيل في الجلسة الحالية
        self.session_attendance_history = []  # قائمة لحفظ معرفات آخر التسجيلات الفردية
        self.session_course_attendance_history = []  # قائمة لحفظ تسجيلات الدورات الكاملة

        # تحديث واجهة البرنامج
        self.update_students_tree()
        self.update_statistics()
        self.update_attendance_display()

        # تطبيق التخطيط المناسب بعد إنشاء كل العناصر
        if self.screen_info["is_small_screen"]:
            self.apply_compact_layout()
        else:
            self.apply_expanded_layout()

        # بدء فحص النشاط - الجزء الجديد
        self.check_inactivity()

    def determine_best_layout(self):
        """تحديد التخطيط الأمثل بناءً على إعدادات الشاشة"""
        screen_width = self.root.winfo_screenwidth()
        screen_height = self.root.winfo_screenheight()

        # حساب حجم النافذة المناسب (90% من حجم الشاشة مع حد أقصى)
        window_width = min(int(screen_width * 0.9), 1400)
        window_height = min(int(screen_height * 0.9), 800)

        # توسيط النافذة
        x = (screen_width - window_width) // 2
        y = (screen_height - window_height) // 2

        # تعيين حجم وموقع النافذة
        self.root.geometry(f"{window_width}x{window_height}+{x}+{y}")

        # تعيين الحد الأدنى لحجم النافذة
        self.root.minsize(800, 600)

        # حفظ معلومات الشاشة لاستخدامها لاحقاً
        self.screen_info = {
            "screen_width": screen_width,
            "screen_height": screen_height,
            "window_width": window_width,
            "window_height": window_height,
            "is_small_screen": screen_width < 1200,
            "is_high_dpi": screen_width > 2000,
            "scale_factor": min(window_width / 1366, window_height / 768)  # عامل القياس النسبي
        }

        # تعديل أحجام الخطوط بناءً على عامل القياس إذا كانت شاشة عالية الدقة
        if self.screen_info["is_high_dpi"]:
            self.adjust_font_sizes(self.screen_info["scale_factor"])

    def setup_styles(self):
        """إعداد أنماط العناصر الرسومية"""
        self.style = ttk.Style()  # ✅ ضروري تعريف الكائن قبل الاستخدام

        self.style.configure("Bold.TNotebook.Tab", font=self.fonts["subtitle"])
        self.style.configure(
            "Bold.Treeview",
            background=self.colors["light"],
            foreground=self.colors["dark"],
            rowheight=30,
            fieldbackground=self.colors["light"],
            font=self.fonts["text_bold"]
        )
        self.style.configure(
            "Bold.Treeview.Heading",
            font=self.fonts["text_bold"],
            background=self.colors["primary"],
            foreground="white"
        )
        self.style.map('Bold.Treeview', background=[('selected', self.colors["primary"])])

        self.style.configure(
            "Profile.Treeview",
            background=self.colors["light"],
            foreground=self.colors["dark"],
            rowheight=32,
            fieldbackground=self.colors["light"],
            font=self.fonts["text_bold"]
        )
        self.style.configure(
            "Profile.Treeview.Heading",
            font=self.fonts["subtitle"],
            background=self.colors["primary"],
            foreground="white"
        )

    def on_window_resize(self, event=None):
        """تستجيب لتغيير حجم النافذة وتعدل العناصر تلقائياً"""
        # تجاهل الأحداث الصغيرة جدًا لتحسين الأداء
        if hasattr(self, 'last_width') and hasattr(self, 'last_height'):
            width_diff = abs(self.root.winfo_width() - self.last_width)
            height_diff = abs(self.root.winfo_height() - self.last_height)
            if width_diff < 10 and height_diff < 10:
                return

        # تخزين الحجم الحالي
        self.last_width = self.root.winfo_width()
        self.last_height = self.root.winfo_height()

        # تحديث معلومات الشاشة
        self.screen_info["window_width"] = self.last_width
        self.screen_info["window_height"] = self.last_height
        self.screen_info["is_small_screen"] = self.last_width < 1200

        # تعديل عرض الأعمدة في الجداول
        self.adjust_column_widths()

        # تعديل حجم النصوص في علامات التبويب
        self.adjust_tab_text()

        # تطبيق التخطيط المناسب
        if self.screen_info["is_small_screen"]:
            self.apply_compact_layout()
        else:
            self.apply_expanded_layout()

    def adjust_font_sizes(self, scale_factor):
        """تعديل أحجام الخطوط بناءً على عامل القياس"""
        # تحديث قيم الخطوط بناءً على عامل القياس
        self.fonts = {
            "large_title": ("Tajawal", int(self.original_fonts["large_title"][1] * scale_factor), "bold"),
            "title": ("Tajawal", int(self.original_fonts["title"][1] * scale_factor), "bold"),
            "subtitle": ("Tajawal", int(self.original_fonts["subtitle"][1] * scale_factor), "bold"),
            "text": ("Tajawal", int(self.original_fonts["text"][1] * scale_factor)),
            "text_bold": ("Tajawal", int(self.original_fonts["text_bold"][1] * scale_factor), "bold"),
            "small": ("Tajawal", int(self.original_fonts["small"][1] * scale_factor))
        }

        # تحديث أنماط العناصر الرسومية
        self.setup_styles()

    def adjust_column_widths(self):
        """تعديل عرض الأعمدة في جداول العرض بناءً على حجم النافذة"""
        try:
            # تعديل جدول سجل الحضور
            if hasattr(self, 'attendance_tree'):
                available_width = self.attendance_tree.winfo_width()
                if available_width > 50:  # تأكد من تهيئة العنصر
                    # تحديد النسب المئوية للأعمدة - زيادة نسبة عمود الاسم
                    col_ratios = [0.12, 0.28, 0.10, 0.12, 0.10, 0.10, 0.10, 0.08]  # زيادة عرض الاسم من 0.20 إلى 0.28

                    # حساب العرض الفعلي لكل عمود
                    for i, ratio in enumerate(col_ratios):
                        width = int(available_width * ratio)
                        if width > 10:  # تجنب القيم السالبة أو الصغيرة جدًا
                            self.attendance_tree.column(self.attendance_tree["columns"][i], width=width)

            # تعديل جدول المتدربين
            if hasattr(self, 'students_tree'):
                available_width = self.students_tree.winfo_width()
                if available_width > 50:
                    col_ratios = [0.15, 0.35, 0.15, 0.15, 0.15, 0.05]  # زيادة عرض الاسم من 0.30 إلى 0.35
                    for i, ratio in enumerate(col_ratios):
                        width = int(available_width * ratio)
                        if width > 10:
                            self.students_tree.column(self.students_tree["columns"][i], width=width)
        except Exception as e:
            print(f"خطأ عند تعديل عرض الأعمدة: {str(e)}")

    def adjust_tab_text(self):
        """تعديل نصوص علامات التبويب حسب المساحة المتاحة"""
        window_width = self.root.winfo_width()

        # على الشاشات الصغيرة، استخدم أسماء مختصرة
        if window_width < 800:
            self.tab_control.tab(0, text="الحضور")
            self.tab_control.tab(1, text="السجل")
            self.tab_control.tab(2, text="المتدربين")
            if self.current_user["permissions"]["is_admin"]:
                self.tab_control.tab(3, text="المستخدمين")
                self.tab_control.tab(4, text="الأرشيف")
            else:
                self.tab_control.tab(3, text="الأرشيف")
        else:
            # على الشاشات الكبيرة، استخدم الأسماء الكاملة
            self.tab_control.tab(0, text="سجل الحضور")
            self.tab_control.tab(1, text="استعراض الحضور")
            self.tab_control.tab(2, text="إدارة المتدربين")
            if self.current_user["permissions"]["is_admin"]:
                self.tab_control.tab(3, text="إدارة المستخدمين")
                self.tab_control.tab(4, text="أرشيف الدورات")
            else:
                self.tab_control.tab(3, text="أرشيف الدورات")

    def apply_compact_layout(self):
        """تطبيق التخطيط المضغوط للشاشات الصغيرة"""
        # تخزين وضع التخطيط الحالي
        self.current_layout = "compact"

        # تنظيم الإحصائيات في عمود واحد
        self.organize_stats_in_one_column()

        # تعديل عدد الأزرار المعروضة
        self.organize_buttons_for_small_screen()

    def apply_expanded_layout(self):
        """تطبيق التخطيط الموسع للشاشات الكبيرة"""
        # تخزين وضع التخطيط الحالي
        self.current_layout = "expanded"

        # تنظيم الإحصائيات في صفين
        self.organize_stats_in_two_rows()

        # عرض كامل للأزرار
        self.show_all_buttons()

    def organize_stats_in_one_column(self):
        """تنظيم بطاقات الإحصائيات في عمود واحد للشاشات الصغيرة"""
        # التنفيذ فقط إذا كان التخطيط الحالي ليس مضغوطًا
        if hasattr(self, 'current_layout') and self.current_layout == "compact":
            return

        if hasattr(self, 'stats_cards') and self.stats_cards:
            stats_frame = self.find_parent_frame(self.stats_cards[0])

            if stats_frame:
                # إزالة الصفوف القديمة
                for child in stats_frame.winfo_children():
                    if child != self.stats_cards[0].master:  # حفظ الإطار الرئيسي
                        child.destroy()

                # إنشاء إطار واحد للعمود
                column_frame = tk.Frame(stats_frame, bg=self.colors["light"])
                column_frame.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)

                # إعادة تنظيم بطاقات الإحصائيات
                for i, card in enumerate(self.stats_cards):
                    card.pack_forget()  # إزالة من التخطيط الحالي
                    card.pack(in_=column_frame, fill=tk.X, padx=5, pady=2)  # إعادة تنظيم في العمود

    def organize_stats_in_two_rows(self):
        """تنظيم بطاقات الإحصائيات في صفين للشاشات الكبيرة"""
        # التنفيذ فقط إذا كان التخطيط الحالي ليس موسعًا
        if hasattr(self, 'current_layout') and self.current_layout == "expanded":
            return

        if hasattr(self, 'stats_cards') and self.stats_cards:
            stats_frame = self.find_parent_frame(self.stats_cards[0])

            if stats_frame:
                # إزالة العمود القديم
                for child in stats_frame.winfo_children():
                    child.destroy()

                # إنشاء إطارين للصفين
                top_counter_frame = tk.Frame(stats_frame, bg=self.colors["light"])
                top_counter_frame.pack(fill=tk.X, padx=5, pady=5)

                bottom_counter_frame = tk.Frame(stats_frame, bg=self.colors["light"])
                bottom_counter_frame.pack(fill=tk.X, padx=5, pady=5)

                # توزيع بطاقات الإحصائيات على الصفين
                half_count = len(self.stats_cards) // 2

                for i, card in enumerate(self.stats_cards):
                    card.pack_forget()  # إزالة من التخطيط الحالي

                    if i < half_count:
                        # الصف الأول
                        card.pack(in_=top_counter_frame, side=tk.RIGHT, padx=5, fill=tk.X, expand=True)
                    else:
                        # الصف الثاني
                        card.pack(in_=bottom_counter_frame, side=tk.RIGHT, padx=5, fill=tk.X, expand=True)

    def find_parent_frame(self, widget):
        """العثور على إطار الأب لعنصر واجهة"""
        if widget is None:
            return None

        parent = widget.master
        while parent is not None:
            if isinstance(parent, tk.LabelFrame) and parent.cget("text") == "إحصائيات اليوم":
                return parent
            parent = parent.master

        return None

    def organize_buttons_for_small_screen(self):
        """تنظيم الأزرار للشاشات الصغيرة"""
        # تنفيذ فقط عند الضرورة
        if hasattr(self, 'current_layout') and self.current_layout == "compact":
            return

        # هنا يمكن تنفيذ تغييرات على تنظيم الأزرار
        # مثل إنشاء قائمة منسدلة لبعض الأزرار الأقل استخداماً
        # أو تصغير حجم الأزرار أو تقليل النص المعروض

        pass  # يمكن تنفيذ المزيد حسب الاحتياج

    def show_all_buttons(self):
        """عرض جميع الأزرار للشاشات الكبيرة"""
        # تنفيذ فقط عند الضرورة
        if hasattr(self, 'current_layout') and self.current_layout == "expanded":
            return

        # إعادة الأزرار إلى حالتها الطبيعية
        # مثل إظهار جميع الأزرار وإعادة النصوص الكاملة

        pass  # يمكن تنفيذ المزيد حسب الاحتياج

    def setup_users_tab(self):
        user_management_frame = tk.Frame(self.users_tab, bg=self.colors["light"])
        user_management_frame.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)

        tk.Label(
            user_management_frame,
            text="إدارة مستخدمي النظام (محمي بكلمة مرور) - خاص بالمشرف",
            font=self.fonts["title"],
            bg=self.colors["primary"],
            fg="white",
            padx=10, pady=10
        ).pack(fill=tk.X)

        open_button = tk.Button(
            user_management_frame,
            text="فتح نافذة إدارة المستخدمين",
            font=self.fonts["text_bold"],
            bg=self.colors["secondary"],
            fg="white",
            padx=20, pady=10, bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=self.protected_open_user_management
        )
        open_button.pack(pady=50)

        # إضافة إطار للنسخ الاحتياطي
        backup_frame = tk.Frame(user_management_frame, bg=self.colors["light"], pady=20)
        backup_frame.pack(pady=20)

        tk.Label(
            backup_frame,
            text="إدارة النسخ الاحتياطية لقاعدة البيانات",
            font=self.fonts["text_bold"],
            bg=self.colors["light"],
            fg=self.colors["dark"]
        ).pack(pady=(0, 10))

        # إضافة أزرار النسخ الاحتياطي والاسترداد
        backup_btn = tk.Button(
            backup_frame,
            text="إنشاء نسخة احتياطية",
            font=self.fonts["text_bold"],
            bg=self.colors["primary"],
            fg="white",
            padx=15, pady=5,
            bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=self.backup_database
        )
        backup_btn.pack(side=tk.LEFT, padx=5)

        restore_btn = tk.Button(
            backup_frame,
            text="استرداد نسخة احتياطية",
            font=self.fonts["text_bold"],
            bg=self.colors["warning"],
            fg="white",
            padx=15, pady=5,
            bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=self.restore_database
        )
        restore_btn.pack(side=tk.LEFT, padx=5)

        optimize_db_btn = tk.Button(
            backup_frame,
            text="تحسين أداء قاعدة البيانات",
            font=self.fonts["text_bold"],
            bg=self.colors["secondary"],
            fg="white",
            padx=15, pady=5,
            bd=0, relief=tk.FLAT,
            cursor="hand2",
            command=self.optimize_database
        )
        optimize_db_btn.pack(side=tk.LEFT, padx=5)

        tk.Label(
            user_management_frame,
            text="لن يتم فتح نافذة إدارة المستخدمين إلا بعد إدخال كلمة مرور المشرف.",
            font=self.fonts["text"],
            bg=self.colors["light"],
            fg=self.colors["dark"],
            padx=10, pady=10, wraplength=700
        ).pack(fill=tk.X)

    def protected_open_user_management(self):
        if not self.current_user["permissions"]["is_admin"]:
            messagebox.showerror("خطأ", "لا تملك صلاحية!")
            return
        admin_pass = simpledialog.askstring("إدخال كلمة المرور", "أدخل كلمة المرور الخاصة بالمشرف:", show='*')
        if not admin_pass:
            return
        cur = self.conn.cursor()
        cur.execute("SELECT password FROM users WHERE username='admin'")
        row = cur.fetchone()
        if not row:
            messagebox.showerror("خطأ", "لا يوجد حساب مشرف رئيسي!")
            return
        admin_real_hash = row[0]
        hashed_input = hashlib.sha256(admin_pass.encode()).hexdigest()
        if hashed_input == admin_real_hash:
            UserManagement(self.root, self.conn, self.current_user, self.colors, self.fonts)
        else:
            messagebox.showerror("خطأ", "كلمة المرور غير صحيحة!")

    def create_tables(self):
        try:
            with self.conn:
                # تعديل جدول المتدربين لإضافة حقول الاستبعاد
                self.conn.execute("""
                    CREATE TABLE IF NOT EXISTS trainees (
                        national_id TEXT PRIMARY KEY,
                        name TEXT,
                        rank TEXT,
                        course TEXT,
                        phone TEXT,
                        is_excluded INTEGER DEFAULT 0,
                        exclusion_reason TEXT DEFAULT '',
                        excluded_date TEXT DEFAULT ''
                    )
                """)

                self.conn.execute("""
                    CREATE TABLE IF NOT EXISTS attendance (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        national_id TEXT,
                        name TEXT,
                        rank TEXT,
                        course TEXT,
                        time TEXT,
                        date TEXT,
                        status TEXT,
                        original_status TEXT,
                        registered_by TEXT,
                        excuse_reason TEXT DEFAULT '',
                        updated_by TEXT,
                        updated_at TEXT,
                        modification_reason TEXT DEFAULT '',
                        receiver_name TEXT DEFAULT ''
                    )
                """)

                # إضافة جدول الفصول
                self.conn.execute("""
                    CREATE TABLE IF NOT EXISTS course_sections (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        course_name TEXT NOT NULL,
                        section_name TEXT NOT NULL,
                        created_date TEXT,
                        UNIQUE(course_name, section_name)
                    )
                """)

                # إضافة جدول تسجيل المتدربين في الفصول
                self.conn.execute("""
                    CREATE TABLE IF NOT EXISTS student_sections (
                        national_id TEXT NOT NULL,
                        course_name TEXT NOT NULL,
                        section_name TEXT NOT NULL,
                        assigned_date TEXT,
                        PRIMARY KEY (national_id, course_name),
                        FOREIGN KEY (national_id) REFERENCES trainees(national_id)
                    )
                """)

                # تحديث جدول معلومات الدورات لإضافة تاريخ النهاية وفئة الدورة
                self.conn.execute("""
                    CREATE TABLE IF NOT EXISTS course_info (
                        course_name TEXT PRIMARY KEY,
                        start_day TEXT,
                        start_month TEXT,
                        start_year TEXT,
                        end_day TEXT,
                        end_month TEXT,
                        end_year TEXT,
                        end_date_system TEXT,  -- تاريخ نهاية الدورة في النظام
                        course_category TEXT,  -- فئة الدورة
                        created_date TEXT
                    )
                """)

                # إضافة جدول المخالفات
                self.conn.execute("""
                    CREATE TABLE IF NOT EXISTS student_violations (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        national_id TEXT,
                        violation_date TEXT,
                        violation_type TEXT,
                        description TEXT,
                        action_taken TEXT,
                        action_date TEXT,
                        recorded_by TEXT,
                        notes TEXT,
                        attachment_path TEXT,
                        FOREIGN KEY (national_id) REFERENCES trainees(national_id)
                    )
                """)

                # جدول جديد لتسجيل التعديلات التاريخية
                self.conn.execute("""
                    CREATE TABLE IF NOT EXISTS historical_edits_log (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        attendance_id INTEGER,
                        national_id TEXT,
                        student_name TEXT,
                        edit_date TEXT,
                        original_date TEXT,
                        old_status TEXT,
                        new_status TEXT,
                        edited_by TEXT,
                        edit_timestamp TEXT,
                        days_difference INTEGER,
                        FOREIGN KEY (attendance_id) REFERENCES attendance(id)
                    )
                """)

                # إضافة الأعمدة الجديدة إذا لم تكن موجودة
                cursor = self.conn.cursor()

                # فحص وإضافة الأعمدة في جدول course_info
                cursor.execute("PRAGMA table_info(course_info)")
                columns = [column[1] for column in cursor.fetchall()]

                if "end_date_system" not in columns:
                    self.conn.execute("ALTER TABLE course_info ADD COLUMN end_date_system TEXT")
                if "course_category" not in columns:
                    self.conn.execute("ALTER TABLE course_info ADD COLUMN course_category TEXT")

                # فحص وإضافة أعمدة الاستبعاد للمتدربين
                cursor.execute("PRAGMA table_info(trainees)")
                columns = [column[1] for column in cursor.fetchall()]

                if "is_excluded" not in columns:
                    self.conn.execute("ALTER TABLE trainees ADD COLUMN is_excluded INTEGER DEFAULT 0")
                if "exclusion_reason" not in columns:
                    self.conn.execute("ALTER TABLE trainees ADD COLUMN exclusion_reason TEXT DEFAULT ''")
                if "excluded_date" not in columns:
                    self.conn.execute("ALTER TABLE trainees ADD COLUMN excluded_date TEXT DEFAULT ''")

                # فحص وإضافة الأعمدة المفقودة في جدول attendance
                cursor.execute("PRAGMA table_info(attendance)")
                columns = [column[1] for column in cursor.fetchall()]

                # إضافة الأعمدة المفقودة إذا لم تكن موجودة
                if "original_status" not in columns:
                    self.conn.execute("ALTER TABLE attendance ADD COLUMN original_status TEXT")
                if "updated_by" not in columns:
                    self.conn.execute("ALTER TABLE attendance ADD COLUMN updated_by TEXT")
                if "updated_at" not in columns:
                    self.conn.execute("ALTER TABLE attendance ADD COLUMN updated_at TEXT")
                if "modification_reason" not in columns:
                    self.conn.execute("ALTER TABLE attendance ADD COLUMN modification_reason TEXT DEFAULT ''")
                if "receiver_name" not in columns:
                    self.conn.execute("ALTER TABLE attendance ADD COLUMN receiver_name TEXT DEFAULT ''")

        except Exception as e:
            messagebox.showerror("خطأ", f"حدث خطأ أثناء إنشاء/تعديل الجداول: {str(e)}")

    def create_indexes(self):
        """إنشاء فهارس لتحسين أداء قاعدة البيانات"""
        try:
            cursor = self.conn.cursor()

            # فهارس للمتدربين
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_trainees_course ON trainees (course)")
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_trainees_name ON trainees (name)")
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_trainees_excluded ON trainees (is_excluded)")

            # فهارس سجلات الحضور
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_attendance_national_id ON attendance (national_id)")
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_attendance_date ON attendance (date)")
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_attendance_status ON attendance (status)")
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_attendance_course ON attendance (course)")
            cursor.execute(
                "CREATE INDEX IF NOT EXISTS idx_attendance_date_national_id ON attendance (date, national_id)")

            # فهارس الفصول
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_sections_course ON course_sections (course_name)")
            cursor.execute(
                "CREATE INDEX IF NOT EXISTS idx_student_sections_national_id ON student_sections (national_id)")
            cursor.execute("CREATE INDEX IF NOT EXISTS idx_student_sections_course ON student_sections (course_name)")

            self.conn.commit()
            print("تم إنشاء الفهارس بنجاح")
        except Exception as e:
            print(f"خطأ في إنشاء الفهارس: {str(e)}")


          
# دالة للتحقق من كلمة المرور
def verify_password(parent, colors, fonts):
    """نافذة التحقق من كلمة المرور"""
    password_window = tk.Toplevel(parent)
    password_window.title("التحقق من الهوية")
    password_window.geometry("400x200")
    password_window.configure(bg=colors["light"])
    password_window.grab_set()

    # توسيط النافذة
    x = (password_window.winfo_screenwidth() - 400) // 2
    y = (password_window.winfo_screenheight() - 200) // 2
    password_window.geometry(f"400x200+{x}+{y}")

    # العنوان
    tk.Label(
        password_window,
        text="الرجاء إدخال كلمة المرور",
        font=fonts["subtitle"],
        bg=colors["light"],
        fg=colors["dark"]
    ).pack(pady=20)

    # إدخال كلمة المرور
    password_var = tk.StringVar()
    password_entry = tk.Entry(
        password_window,
        textvariable=password_var,
        font=fonts["text"],
        show="*",
        width=20,
        justify=tk.CENTER
    )
    password_entry.pack(pady=10)
    password_entry.focus()

    result = {'success': False}

    def check_password():
        if password_var.get() == "123456":
            result['success'] = True
            password_window.destroy()
        else:
            messagebox.showerror("خطأ", "كلمة المرور غير صحيحة", parent=password_window)
            password_entry.delete(0, tk.END)
            password_entry.focus()

    def on_enter(event):
        check_password()

    password_entry.bind('<Return>', on_enter)

    # إطار الأزرار
    button_frame = tk.Frame(password_window, bg=colors["light"])
    button_frame.pack(pady=20)

    tk.Button(
        button_frame,
        text="تأكيد",
        font=fonts["text_bold"],
        bg=colors["success"],
        fg="white",
        padx=20,
        pady=5,
        bd=0,
        relief=tk.FLAT,
        cursor="hand2",
        command=check_password
    ).pack(side=tk.LEFT, padx=5)

    tk.Button(
        button_frame,
        text="إلغاء",
        font=fonts["text_bold"],
        bg=colors["danger"],
        fg="white",
        padx=20,
        pady=5,
        bd=0,
        relief=tk.FLAT,
        cursor="hand2",
        command=password_window.destroy
    ).pack(side=tk.LEFT, padx=5)

    # انتظار إغلاق النافذة
    password_window.wait_window()

    return result['success']


# دالة محدثة لإضافة الأيقونة في التطبيق الرئيسي مع كلمة المرور
def add_absence_monitoring_icon(self):
    """إضافة أيقونة نظام مراقبة الغياب في التطبيق الرئيسي"""
    # إضافة تبويب جديد في نافذة التطبيق الرئيسية
    if hasattr(self, 'tab_control'):
        self.absence_monitor_tab = tk.Frame(self.tab_control, bg=self.colors["light"])
        self.tab_control.add(self.absence_monitor_tab, text="مراقبة الغياب")

        # إطار للمحتوى
        content_frame = tk.Frame(self.absence_monitor_tab, bg=self.colors["light"])
        content_frame.pack(fill=tk.BOTH, expand=True, padx=20, pady=20)

        # عنوان
        tk.Label(
            content_frame,
            text="نظام مراقبة الغياب والإنذارات",
            font=self.fonts["title"],
            bg=self.colors["light"],
            fg=self.colors["primary"]
        ).pack(pady=20)

        # وصف
        tk.Label(
            content_frame,
            text="نظام متقدم لمراقبة غياب المتدربين وفقاً لتعليمات التدريب المستديمة\n"
                 "يحسب الحد المسموح للغياب حسب مدة الدورة ويُصدر إنذارات تلقائية",
            font=self.fonts["text"],
            bg=self.colors["light"],
            justify=tk.CENTER
        ).pack(pady=10)

        # دالة لفتح النظام بعد التحقق من كلمة المرور
        def open_absence_system():
            if verify_password(self.root, self.colors, self.fonts):
                AbsenceMonitoringSystem(self.root, self, self.colors, self.fonts)

        # زر فتح النظام
        tk.Button(
            content_frame,
            text="فتح نظام مراقبة الغياب",
            font=self.fonts["text_bold"],
            bg=self.colors["primary"],
            fg="white",
            padx=30,
            pady=15,
            bd=0,
            relief=tk.FLAT,
            cursor="hand2",
            command=open_absence_system
        ).pack(pady=20)


# =============================================================================
#                      نقطة التشغيل الرئيسية
# =============================================================================
if __name__ == "__main__":
    # التحقق من الترخيص قبل بدء البرنامج
    verify_license()

    # البدء في تشغيل البرنامج
    root = tk.Tk()
    LoginSystem(root)
    root.mainloop()
