تمام! هذا هو الملف الكامل shdin_bank.py جاهز للنسخ والتشغيل مباشرة (يدعم الإيداع/السحب/التحويل + حفظ JSON + محاكاة سوق ShDin مقابل الدولار + رسوم بيانية):

# -*- coding: utf-8 -*-
import tkinter as tk
from tkinter import messagebox, simpledialog
import json
import os
import datetime
from enum import Enum
import random
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg

# ----------- حسابات وأنواعها -----------
class AccountType(Enum):
    جاري = "جاري"
    توفير = "توفير"
    استثماري = "استثماري"

class Account:
    def __init__(self, account_number, account_type: AccountType, balance=0.0):
        self.account_number = account_number
        self.account_type = account_type
        self.balance = balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("الرجاء إدخال مبلغ أكبر من صفر")
        self.balance += amount

    def withdraw(self, amount):
        if amount <= 0:
            raise ValueError("الرجاء إدخال مبلغ أكبر من صفر")
        if amount > self.balance:
            raise ValueError("الرصيد غير كافٍ")
        self.balance -= amount

# ----------- العمليات -----------
class Transaction:
    def __init__(self, acc_from, acc_to, amount, tx_type, date=None):
        self.acc_from = acc_from          # رقم الحساب المرسل (أو None للإيداع)
        self.acc_to = acc_to              # رقم الحساب المستلم (أو None للسحب)
        self.amount = amount
        self.tx_type = tx_type            # "إيداع" | "سحب" | "تحويل"
        self.date = date or datetime.datetime.now().isoformat()

# ----------- نظام البنك -----------
class BankSystem:
    def __init__(self, accounts_file='accounts.json', transactions_file='transactions.json'):
        self.accounts_file = accounts_file
        self.transactions_file = transactions_file
        self.accounts = self.load_accounts()
        self.transactions = self.load_transactions()

        # مؤشرات الاقتصاد المحلي + سوق ShDin
        self.exchange_rate = 100          # سعر الصرف المحلي (وحدة افتراضية)
        self.inflation = 2.0              # التضخم %
        self.liquidity = 1_000_000        # السيولة المحلية
        self.market_rate = 1.00           # سعر ShDin مقابل الدولار (ShDin/USD)

        # تاريخ المؤشرات للرسم
        self.history_rate = [self.exchange_rate]
        self.history_inflation = [self.inflation]
        self.history_liquidity = [self.liquidity]
        self.history_market = [self.market_rate]

    # --- تخزين/تحميل ---
    def load_accounts(self):
        if os.path.exists(self.accounts_file):
            with open(self.accounts_file, "r", encoding="utf-8") as f:
                data = json.load(f)
                return {
                    acc['account_number']: Account(
                        acc['account_number'],
                        AccountType(acc['account_type']),
                        acc['balance']
                    ) for acc in data
                }
        return {}

    def save_accounts(self):
        with open(self.accounts_file, "w", encoding="utf-8") as f:
            data = [{
                "account_number": acc.account_number,
                "account_type": acc.account_type.value,
                "balance": acc.balance
            } for acc in self.accounts.values()]
            json.dump(data, f, ensure_ascii=False, indent=2)

    def load_transactions(self):
        if os.path.exists(self.transactions_file):
            with open(self.transactions_file, "r", encoding="utf-8") as f:
                return json.load(f)
        return []

    def save_transactions(self):
        with open(self.transactions_file, "w", encoding="utf-8") as f:
            json.dump(self.transactions, f, ensure_ascii=False, indent=2)

    # --- تحديث المؤشرات بعد كل عملية + تقلبات السوق العالمية ---
    def update_economy(self, transaction_type, amount):
        # تأثير محلي مباشر
        if transaction_type == "deposit":
            self.exchange_rate -= amount * 0.05
            self.liquidity += amount * 0.01
            self.inflation = max(0, self.inflation - 0.02)
            self.market_rate -= amount * 0.0005
        elif transaction_type == "withdraw":
            self.exchange_rate += amount * 0.05
            self.liquidity = max(0, self.liquidity - amount * 0.01)
            self.inflation += 0.03
            self.market_rate += amount * 0.0005
        elif transaction_type == "transfer":
            # تأثير أخف + اهتزاز بسيط
            self.exchange_rate += random.uniform(-10, 10)
            self.liquidity += random.uniform(-2, 2)
            self.inflation += random.uniform(-0.05, 0.05)
            self.market_rate += random.uniform(-0.001, 0.001)

        # تقلبات سوق عالمية عشوائية (تحاكي الأخبار والسيولة العالمية)
        global_fluct = random.uniform(-0.02, 0.02)  # ±2%
        self.market_rate *= (1 + global_fluct)

        # قيود وحدود منطقية
        self.exchange_rate = float(max(50, min(self.exchange_rate, 3000)))
        self.inflation = float(max(0, min(self.inflation, 25)))
        self.liquidity = float(max(100, min(self.liquidity, 2_000_000)))
        self.market_rate = float(max(0.5, min(self.market_rate, 3.0)))  # بين 0.5 و 3 ShDin لكل دولار

        # تحديث التاريخ
        self.history_rate.append(self.exchange_rate)
        self.history_inflation.append(self.inflation)
        self.history_liquidity.append(self.liquidity)
        self.history_market.append(self.market_rate)

    # --- عمليات البنك ---
    def deposit(self, account_number, amount):
        if account_number not in self.accounts:
            raise ValueError("رقم الحساب غير موجود!")
        self.accounts[account_number].deposit(amount)
        self.transactions.append(Transaction(None, account_number, amount, "إيداع").__dict__)
        self.update_economy("deposit", amount)
        self.save_accounts()
        self.save_transactions()

    def withdraw(self, account_number, amount):
        if account_number not in self.accounts:
            raise ValueError("رقم الحساب غير موجود!")
        self.accounts[account_number].withdraw(amount)
        self.transactions.append(Transaction(account_number, None, amount, "سحب").__dict__)
        self.update_economy("withdraw", amount)
        self.save_accounts()
        self.save_transactions()

    def transfer(self, from_acc, to_acc, amount):
        if from_acc not in self.accounts or to_acc not in self.accounts:
            raise ValueError("رقم حساب غير صحيح!")
        self.accounts[from_acc].withdraw(amount)
        self.accounts[to_acc].deposit(amount)
        self.transactions.append(Transaction(from_acc, to_acc, amount, "تحويل").__dict__)
        self.update_economy("transfer", amount)
        self.save_accounts()
        self.save_transactions()

    def report(self):
        accounts_summary = [
            f"{acc.account_number}: {acc.account_type.value} - الرصيد: {acc.balance:.2f}"
            for acc in self.accounts.values()
        ]
        last_transactions = [
            f"{tx['date']} | {tx['tx_type']} | من: {tx['acc_from']} إلى: {tx['acc_to']} | المبلغ: {tx['amount']}"
            for tx in self.transactions[-10:]
        ]
        return accounts_summary, last_transactions

    def add_account(self, account_number, account_type, balance=0.0):
        self.accounts[account_number] = Account(account_number, AccountType(account_type), balance)
        self.save_accounts()

# ----------- ألوان الواجهة -----------
DARK_BG = "#232946"
BRIGHT_ACCENT = "#00B3FF"   # أزرق لمّاع
BRIGHT_SECOND = "#9AA0A6"   # رمادي فاتح
PURPLE = "#6A5ACD"
TEXT_MAIN = "#EAEFF2"

# ----------- واجهة المستخدم -----------
class BankApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("بنك ShDin - محاكاة سوق العملات")
        self.configure(bg=DARK_BG)
        self.geometry("820x920")

        self.bank = BankSystem()
        if not self.bank.accounts:
            self.bank.add_account("10001", "جاري", 1000)
            self.bank.add_account("10002", "توفير", 2000)
            self.bank.add_account("10003", "استثماري", 3000)

        self.create_widgets()
        self.refresh_report()
        self.plot_economy()

    def create_widgets(self):
        # عنوان
        title = tk.Label(self, text="💙 بنك ShDin - عملة شيدن", fg=BRIGHT_ACCENT, bg=DARK_BG,
                         font=("Arial", 22, "bold"))
        title.pack(pady=10)

        # شريط أزرار
        btn_frame = tk.Frame(self, bg=DARK_BG)
        btn_frame.pack(pady=10)

        def styled_btn(parent, text, bg, cmd, col, row):
            b = tk.Button(parent, text=text, width=16, bg=bg, fg="#101418",
                          font=("Arial", 14, "bold"), relief="flat", command(cmd))
            b.grid(row=row, column=col, padx=8, pady=6)
            return b

        styled_btn(btn_frame, "إيداع", BRIGHT_ACCENT, self.deposit_ui, 0, 0)
        styled_btn(btn_frame, "سحب", "#FFD166", self.withdraw_ui, 1, 0)
        styled_btn(btn_frame, "تحويل", "#06D6A0", self.transfer_ui, 0, 1)
        styled_btn(btn_frame, "تقرير العمليات", "#7CFC00", self.refresh_report, 1, 1)

        # تقرير نصّي
        self.report_label = tk.Label(self, text="", fg=TEXT_MAIN, bg=DARK_BG,
                                     font=("Arial", 12), justify="right", anchor="e")
        self.report_label.pack(fill="both", expand=False, padx=16, pady=12)

        # منطقة الرسم البياني
        self.fig, self.axs = plt.subplots(2, 2, figsize=(8, 6))
        self.fig.patch.set_facecolor('#232946')
        plt.tight_layout()
        self.canvas = FigureCanvasTkAgg(self.fig, master=self)
        self.canvas.get_tk_widget().pack(fill="both", expand=True, padx=12, pady=12)

        # شريط سفلي
        footer = tk.Label(self, text="© بنك شيدن 2025 — محاكاة تعليمية",
                          fg=BRIGHT_SECOND, bg=DARK_BG, font=("Arial", 10))
        footer.pack(side="bottom", pady=10)

    def refresh_report(self):
        accounts_summary, last_transactions = self.bank.report()
        text = "📚 الحسابات:\n" + "\n".join(accounts_summary)
        text += "\n\n🧾 آخر العمليات:\n" + ("\n".join(last_transactions) if last_transactions else "لا توجد عمليات بعد")
        text += (
            f"\n\n📊 مؤشرات اقتصادية حالية:"
            f"\n• سعر الصرف المحلي: {self.bank.exchange_rate:.2f}"
            f"\n• التضخم: {self.bank.inflation:.2f}%"
            f"\n• السيولة: {int(self.bank.liquidity):,}"
            f"\n• سعر ShDin مقابل الدولار (ShDin/USD): {self.bank.market_rate:.2f}"
        )
        self.report_label.config(text=text)
        self.plot_economy()

    def plot_economy(self):
        axs = self.axs.flatten()

        # سعر الصرف
        axs[0].clear()
        axs[0].plot(self.bank.history_rate)
        axs[0].set_title('سعر الصرف المحلي', color='white')

        # التضخم
        axs[1].clear()
        axs[1].plot(self.bank.history_inflation)
        axs[1].set_title('التضخم %', color='white')

        # السيولة
        axs[2].clear()
        axs[2].plot(self.bank.history_liquidity)
        axs[2].set_title('السيولة', color='white')

        # ShDin مقابل الدولار
        axs[3].clear()
        axs[3].plot(self.bank.history_market)
        axs[3].set_title('سعر ShDin مقابل الدولار (ShDin/USD)', color='white')

        # تنسيق عام
        for ax in axs:
            ax.tick_params(colors='white')
            ax.set_facecolor('#232946')
            ax.title.set_fontsize(10)

        self.canvas.draw()

    # --- نوافذ العمليات ---
    def deposit_ui(self):
        acc_num = simpledialog.askstring("إيداع", "رقم الحساب:")
        amount = simpledialog.askfloat("إيداع", "المبلغ:")
        if not acc_num or amount is None:
            return
        try:
            self.bank.deposit(acc_num, float(amount))
            messagebox.showinfo("نجاح", f"تم الإيداع بنجاح في الحساب {acc_num}")
            self.refresh_report()
        except Exception as e:
            messagebox.showerror("خطأ", str(e))

    def withdraw_ui(self):
        acc_num = simpledialog.askstring("سحب", "رقم الحساب:")
        amount = simpledialog.askfloat("سحب", "المبلغ:")
        if not acc_num or amount is None:
            return
        try:
            self.bank.withdraw(acc_num, float(amount))
            messagebox.showinfo("نجاح", f"تم السحب بنجاح من الحساب {acc_num}")
            self.refresh_report()
        except Exception as e:
            messagebox.showerror("خطأ", str(e))

    def transfer_ui(self):
        acc_from = simpledialog.askstring("تح

