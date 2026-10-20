@echo off
chcp 65001 >nul
cd /d "%~dp0"
title PhoneLink Kurulum

python --version >nul 2>nul
if errorlevel 1 (
  echo Python kuruluyor, lutfen bekle...
  winget install -e --id Python.Python.3.12 --accept-package-agreements --accept-source-agreements
  echo.
  echo Python kuruldu. Bu pencereyi kapat ve bu dosyaya TEKRAR cift tikla.
  pause
  exit /b
)

python -m pip install pynput -q

if not exist "scrcpy\scrcpy.exe" (
  echo scrcpy indiriliyor, lutfen bekle...
  powershell -NoProfile -Command "[Net.ServicePointManager]::SecurityProtocol='Tls12'; $r=Invoke-RestMethod 'https://api.github.com/repos/Genymobile/scrcpy/releases/latest'; $a=$r.assets | Where-Object { $_.name -like 'scrcpy-win64-*.zip' } | Select-Object -First 1; Invoke-WebRequest $a.browser_download_url -OutFile scrcpy.zip; Expand-Archive scrcpy.zip -DestinationPath scrcpy_tmp -Force; Remove-Item scrcpy.zip"
  for /d %%D in (scrcpy_tmp\*) do move "%%D" scrcpy >nul
  rmdir /s /q scrcpy_tmp 2>nul
)

if not exist "scrcpy\scrcpy.exe" (
  echo scrcpy indirilemedi. Internet baglantini kontrol edip tekrar dene.
  pause
  exit /b
)

powershell -NoProfile -Command "$t=[IO.File]::ReadAllText('%~f0',[Text.Encoding]::UTF8); $p=$t.Substring($t.IndexOf('#__'+'PY__')); [IO.File]::WriteAllText('%~dp0phonelink.py',$p,(New-Object Text.UTF8Encoding $false))"

set "PATH=%~dp0scrcpy;%PATH%"
start "" pythonw "%~dp0phonelink.py"
exit
#__PY__
#!/usr/bin/env python3
"""PhoneLink v2 - Telefon ekranını bilgisayara yansıt, kablosuz bağlan, tuş ata."""
import base64, json, math, os, re, shutil, struct, subprocess, threading, time
import tkinter as tk
from tkinter import ttk, messagebox

try:
    from pynput import keyboard
except ImportError:
    keyboard = None

CFG = os.path.join(os.path.expanduser("~"), ".phonelink.json")
ADB = shutil.which("adb") or "adb"
SCRCPY = shutil.which("scrcpy") or "scrcpy"
NOWIN = 0x08000000 if os.name == "nt" else 0

BG, CARD, CARD2 = "#0e1410", "#18211b", "#223027"
GREEN, TEXT, MUTED, RED = "#3DDC84", "#e8f3ec", "#8ba396", "#ff6b6b"
F = ("Segoe UI", 10)
FB = ("Segoe UI", 10, "bold")

KEYEV = {"back": 4, "home": 3, "recents": 187, "vol_up": 24, "vol_down": 25, "power": 26}
SWIPE = {"swipe_up": (0, -400), "swipe_down": (0, 400),
         "swipe_left": (-400, 0), "swipe_right": (400, 0)}
ACTIONS = ["tap", *SWIPE, *KEYEV]
ALIAS = {"return": "enter", "prior": "page_up", "next": "page_down",
         "escape": "esc", "control": "ctrl"}


def norm(name):
    n = str(name).lower()
    for s in ("_l", "_r"):
        if n.endswith(s):
            n = n[:-2]
    return ALIAS.get(n, n)


def adb(*args):
    try:
        return subprocess.run([ADB, *args], capture_output=True, timeout=20,
                              creationflags=NOWIN).stdout
    except Exception:
        return b""


def adbt(*args):
    try:
        r = subprocess.run([ADB, *args], stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                           timeout=25, creationflags=NOWIN)
        return r.stdout.decode(errors="ignore")
    except FileNotFoundError:
        return "adb bulunamadı (scrcpy klasörü eksik)."
    except subprocess.TimeoutExpired:
        return "Zaman aşımı."


class Shell:
    p = None

    def start(self):
        self.stop()
        self.p = subprocess.Popen([ADB, "shell"], stdin=subprocess.PIPE,
                                  stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
                                  creationflags=NOWIN)

    def stop(self):
        if self.p and self.p.poll() is None:
            self.p.terminate()
        self.p = None

    def send(self, cmd):
        if self.p and self.p.poll() is None:
            self.p.stdin.write((cmd + "\n").encode())
            self.p.stdin.flush()


# ---------- görsel parçalar ----------
class Btn(tk.Label):
    def __init__(self, parent, text, cmd, primary=False):
        super().__init__(parent, text=text, font=FB, padx=14, pady=8, cursor="hand2")
        self.cmd = cmd
        self.style(primary)
        self.bind("<Button-1>", lambda e: self.cmd())
        self.bind("<Enter>", lambda e: self.config(bg=self.hov))
        self.bind("<Leave>", lambda e: self.config(bg=self.base))

    def style(self, primary):
        self.base = GREEN if primary else CARD2
        self.hov = "#6ff0a6" if primary else "#2f4237"
        self.config(bg=self.base, fg="#04210f" if primary else TEXT)


def logo(parent, bg):
    c = tk.Canvas(parent, width=56, height=36, bg=bg, highlightthickness=0)
    c.create_line(17, 10, 10, 2, fill=GREEN, width=3, capstyle="round")
    c.create_line(39, 10, 46, 2, fill=GREEN, width=3, capstyle="round")
    c.create_arc(6, 6, 50, 50, start=0, extent=180, fill=GREEN, outline=GREEN)
    c.create_oval(16, 19, 22, 25, fill=bg, outline=bg)
    c.create_oval(34, 19, 40, 25, fill=bg, outline=bg)
    return c


def card(parent, title):
    o = tk.Frame(parent, bg=CARD)
    o.pack(fill="x", pady=5)
    tk.Label(o, text=title, bg=CARD, fg=GREEN, font=FB).pack(anchor="w", padx=12, pady=(10, 2))
    i = tk.Frame(o, bg=CARD)
    i.pack(fill="x", padx=12, pady=(0, 12))
    return i


def note(parent, text):
    tk.Label(parent, text=text, bg=CARD, fg=MUTED, font=("Segoe UI", 9),
             wraplength=400, justify="left").pack(anchor="w", pady=(0, 6))


def row(parent):
    r = tk.Frame(parent, bg=CARD)
    r.pack(fill="x", pady=3)
    return r


def entry(parent, var, w=22):
    return tk.Entry(parent, textvariable=var, width=w, bg=CARD2, fg=TEXT, font=F,
                    insertbackground=TEXT, relief="flat", highlightthickness=1,
                    highlightbackground=CARD2, highlightcolor=GREEN)


def label(parent, text):
    return tk.Label(parent, text=text, bg=CARD, fg=TEXT, font=F)


def check(parent, text, var):
    return tk.Checkbutton(parent, text=text, variable=var, bg=CARD, fg=TEXT, font=F,
                          selectcolor=CARD2, activebackground=CARD, activeforeground=TEXT)


# ---------- tuş ekleme penceresi ----------
class AddDialog(tk.Toplevel):
    def __init__(self, app):
        super().__init__(app.root, bg=BG)
        self.app, self.key, self.pt, self.scale = app, None, None, 1
        self.title("Tuş ekle")
        top = tk.Frame(self, bg=BG)
        top.pack(fill="x", padx=10, pady=10)
        self.keylbl = tk.Label(top, text="Tuş: —", bg=BG, fg=TEXT, font=FB, width=14, anchor="w")
        self.keylbl.pack(side="left")
        Btn(top, "Tuşa bas", self.listen).pack(side="left", padx=4)
        self.act = ttk.Combobox(top, values=ACTIONS, state="readonly", width=11)
        self.act.set("tap")
        self.act.pack(side="left", padx=4)
        Btn(top, "Kaydet", self.save, True).pack(side="right")
        Btn(top, "Yenile", self.shot).pack(side="right", padx=4)
        tk.Label(self, text="tap/kaydırma için telefon görüntüsünde bir yere tıkla "
                 "(back/home/vol gibi aksiyonlarda gerekmez)",
                 bg=BG, fg=MUTED, font=("Segoe UI", 9)).pack()
        self.cv = tk.Canvas(self, width=360, height=640, bg="#000", highlightthickness=0)
        self.cv.pack(padx=10, pady=10)
        self.cv.bind("<Button-1>", self.click)
        self.shot()

    def listen(self):
        self.keylbl.config(text="Bir tuşa bas…", fg=GREEN)
        self.bind("<KeyPress>", self.onkey)
        self.focus_force()

    def onkey(self, e):
        self.key = norm(e.keysym)
        self.keylbl.config(text=f"Tuş: {self.key}", fg=TEXT)
        self.unbind("<KeyPress>")

    def shot(self):
        data = adb("exec-out", "screencap", "-p")
        if not data.startswith(b"\x89PNG"):
            messagebox.showerror("Hata", "Ekran görüntüsü alınamadı. Telefon bağlı mı?", parent=self)
            return
        w, h = struct.unpack(">II", data[16:24])
        img = tk.PhotoImage(data=base64.b64encode(data)).subsample(max(1, math.ceil(h / 680)))
        self.img, self.scale = img, w / img.width()
        self.cv.config(width=img.width(), height=img.height())
        self.cv.delete("all")
        self.cv.create_image(0, 0, anchor="nw", image=img)

    def click(self, e):
        self.pt = (int(e.x * self.scale), int(e.y * self.scale))
        self.cv.delete("mark")
        self.cv.create_oval(e.x - 12, e.y - 12, e.x + 12, e.y + 12, outline=GREEN,
                            width=3, tags="mark")

    def save(self):
        a = self.act.get()
        if not self.key or (a not in KEYEV and not self.pt):
            messagebox.showwarning("Eksik", "Tuş ve (gerekiyorsa) konum seç.", parent=self)
            return
        x, y = self.pt or (0, 0)
        self.app.maps = [m for m in self.app.maps if m["key"] != self.key]
        self.app.maps.append({"key": self.key, "action": a, "x": x, "y": y})
        self.app.changed()
        self.destroy()


# ---------- ana uygulama ----------
class App:
    def __init__(self):
        self.root = r = tk.Tk()
        r.title("PhoneLink")
        r.configure(bg=BG)
        r.geometry("500x760")
        r.minsize(460, 660)
        self.shell, self.mirror, self.listener = Shell(), None, None
        self.down, self.keys_on = set(), False
        self.cfg = self.load()
        self.maps = self.cfg.get("maps", [])
        self.v = {"size": tk.StringVar(value="1024"), "bitrate": tk.StringVar(value="8"),
                  "top": tk.BooleanVar(), "awake": tk.BooleanVar(value=True),
                  "off": tk.BooleanVar(), "nokey": tk.BooleanVar(value=True),
                  "addr": tk.StringVar(value=self.cfg.get("ip", "")),
                  "pair": tk.StringVar(), "code": tk.StringVar()}
        self.theme()
        self.log = tk.Label(r, text="Hazır.", bg=CARD, fg=MUTED, font=("Segoe UI", 9),
                            anchor="w", padx=12, pady=6, wraplength=470, justify="left")
        self.log.pack(side="bottom", fill="x")
        self.header()
        self.tabs()
        self.show("conn")
        self.fill()
        self.poll()
        r.protocol("WM_DELETE_WINDOW", self.quit)

    def theme(self):
        s = ttk.Style()
        s.theme_use("clam")
        s.configure("Treeview", background=CARD, fieldbackground=CARD, foreground=TEXT,
                    rowheight=28, borderwidth=0, font=F)
        s.configure("Treeview.Heading", background=CARD2, foreground=GREEN,
                    relief="flat", font=("Segoe UI", 9, "bold"))
        s.map("Treeview", background=[("selected", GREEN)], foreground=[("selected", "#04210f")])
        s.configure("TCombobox", fieldbackground=CARD2, background=CARD2, foreground=TEXT,
                    arrowcolor=GREEN, borderwidth=0)

    def header(self):
        h = tk.Frame(self.root, bg=BG)
        h.pack(fill="x", padx=16, pady=(14, 6))
        logo(h, BG).pack(side="left")
        t = tk.Frame(h, bg=BG)
        t.pack(side="left", padx=10)
        tk.Label(t, text="PhoneLink", bg=BG, fg=TEXT, font=("Segoe UI", 17, "bold")).pack(anchor="w")
        tk.Label(t, text="Telefonunu bilgisayara bağla", bg=BG, fg=MUTED,
                 font=("Segoe UI", 9)).pack(anchor="w")
        self.pill = tk.Label(h, text="○ Bağlı değil", bg=CARD, fg=RED, font=FB, padx=12, pady=5)
        self.pill.pack(side="right")

    def tabs(self):
        bar = tk.Frame(self.root, bg=BG)
        bar.pack(fill="x", padx=16)
        self.tabl = {}
        for k, t in (("conn", "Bağlantı"), ("mirror", "Yansıtma"), ("keys", "Tuşlar")):
            l = tk.Label(bar, text=t, bg=BG, fg=MUTED, font=("Segoe UI", 11, "bold"),
                         padx=14, pady=8, cursor="hand2")
            l.pack(side="left")
            l.bind("<Button-1>", lambda e, k=k: self.show(k))
            self.tabl[k] = l
        tk.Frame(self.root, bg=CARD2, height=1).pack(fill="x", padx=16)
        body = tk.Frame(self.root, bg=BG)
        body.pack(fill="both", expand=True, padx=16, pady=6)
        self.pages = {k: tk.Frame(body, bg=BG) for k in self.tabl}
        self.page_conn()
        self.page_mirror()
        self.page_keys()

    def show(self, k):
        for n, p in self.pages.items():
            p.pack_forget()
            self.tabl[n].config(fg=GREEN if n == k else MUTED)
        self.pages[k].pack(fill="both", expand=True)

    # ----- sayfa: bağlantı -----
    def page_conn(self):
        p = self.pages["conn"]
        c = card(p, "1 · USB ile bağlan")
        note(c, "Telefonda Geliştirici seçenekleri > USB hata ayıklamayı aç, kabloyu tak ve "
                "çıkan izin sorusuna 'İzin ver' de.")
        Btn(c, "Cihazı ara", self.check_now).pack(anchor="w")

        c = card(p, "2 · Kablosuza geç (en kolay)")
        note(c, "Telefon USB ile bağlıyken bas. IP otomatik bulunur ve bağlanır, sonra "
                "kabloyu çıkarabilirsin. Telefon ve bilgisayar aynı Wi-Fi'de olmalı.")
        Btn(c, "Kablosuza geç", self.to_wifi, True).pack(anchor="w")

        c = card(p, "3 · Kodla eşleştir (Android 11+, kablosuz)")
        note(c, "Telefonda: Kablosuz hata ayıklama > 'Eşleştirme koduyla cihazı eşle'. "
                "Ekrandaki IP:port ve 6 haneli kodu yaz. Bu sadece bir kez gerekir.")
        r = row(c)
        label(r, "Eşleştirme IP:port").pack(side="left")
        entry(r, self.v["pair"]).pack(side="right")
        r = row(c)
        label(r, "Kod").pack(side="left")
        entry(r, self.v["code"]).pack(side="right")
        Btn(c, "Eşleştir", self.pair).pack(anchor="w", pady=(4, 8))
        note(c, "Sonra kablosuz hata ayıklama ana ekranındaki IP:port'u gir:")
        r = row(c)
        entry(r, self.v["addr"], 26).pack(side="left")
        Btn(r, "Bağlan", self.connect).pack(side="right")

    # ----- sayfa: yansıtma -----
    def page_mirror(self):
        p = self.pages["mirror"]
        c = card(p, "Yansıtma ayarları")
        r = row(c)
        label(r, "Çözünürlük").pack(side="left")
        ttk.Combobox(r, textvariable=self.v["size"], width=6,
                     values=["0", "720", "1024", "1920"]).pack(side="left", padx=(6, 16))
        label(r, "Bitrate (Mbps)").pack(side="left")
        entry(r, self.v["bitrate"], 5).pack(side="left", padx=6)
        for k, t in (("top", "Pencere hep üstte"), ("awake", "Telefon ekranı uyanık kalsın"),
                     ("off", "Telefon ekranını kapat (pil tasarrufu)"),
                     ("nokey", "scrcpy klavyesini kapat (tuş eşleme için)")):
            check(c, t, self.v[k]).pack(anchor="w")
        self.mbtn = Btn(c, "▶  Yansıtmayı başlat", self.toggle_mirror, True)
        self.mbtn.pack(fill="x", pady=(10, 0))

        c = card(p, "Hızlı tuşlar")
        g = tk.Frame(c, bg=CARD)
        g.pack(fill="x")
        items = [("Geri", "back"), ("Ana ekran", "home"), ("Son uygulamalar", "recents"),
                 ("Ses +", "vol_up"), ("Ses −", "vol_down"), ("Güç", "power")]
        for i, (t, k) in enumerate(items):
            Btn(g, t, lambda k=k: self.keyev(k)).grid(row=i // 3, column=i % 3,
                                                       padx=3, pady=3, sticky="ew")
            g.columnconfigure(i % 3, weight=1)
        Btn(g, "Ekran görüntüsü al (Masaüstüne kaydet)", self.snap).grid(
            row=2, column=0, columnspan=3, padx=3, pady=3, sticky="ew")

    # ----- sayfa: tuşlar -----
    def page_keys(self):
        c = card(self.pages["keys"], "Tuş atamaları")
        self.tree = ttk.Treeview(c, columns=("key", "action", "x", "y"), show="headings", height=9)
        for col, w in zip(("key", "action", "x", "y"), (90, 120, 60, 60)):
            self.tree.heading(col, text=col.upper())
            self.tree.column(col, width=w, anchor="center")
        self.tree.pack(fill="x", pady=(0, 8))
        r = row(c)
        Btn(r, "+ Ekle", lambda: AddDialog(self), True).pack(side="left")
        Btn(r, "Sil", self.delete).pack(side="left", padx=6)
        self.kbtn = Btn(r, "Tuş eşleme: KAPALI", self.toggle_keys)
        self.kbtn.pack(side="right")
        note(c, "Açıkken atanan tuşlar bilgisayarın her yerinde çalışır, işin bitince kapat.")

    # ----- yardımcılar -----
    def bg(self, fn):
        threading.Thread(target=fn, daemon=True).start()

    def say(self, msg, ok=True):
        self.root.after(0, lambda: self.log.config(text=msg, fg=GREEN if ok else RED))

    def devices(self):
        out = adbt("devices").splitlines()[1:]
        return [l.split()[0] for l in out if l.strip().endswith("device")]

    def poll(self):
        def w():
            d = self.devices()
            self.root.after(0, lambda: self.pill.config(
                text=f"●  {d[0]}" if d else "○ Bağlı değil", fg=GREEN if d else RED))
        self.bg(w)
        self.root.after(3000, self.poll)

    def check_now(self):
        def w():
            d = self.devices()
            self.say("Bulundu: " + ", ".join(d) if d else
                     "Cihaz yok. Kabloyu, USB hata ayıklamayı ve izin sorusunu kontrol et.", bool(d))
        self.bg(w)

    def to_wifi(self):
        def w():
            if not [x for x in self.devices() if ":" not in x]:
                return self.say("Önce telefonu USB ile bağla.", False)
            self.say("IP adresi alınıyor…")
            out = adbt("shell", "ip", "route")
            lines = [l for l in out.splitlines() if "wlan" in l] or [out]
            m = re.search(r"src (\d+\.\d+\.\d+\.\d+)", "\n".join(lines))
            if not m:
                return self.say("Wi-Fi IP'si bulunamadı. Telefon Wi-Fi'ye bağlı mı?", False)
            addr = m.group(1) + ":5555"
            adbt("tcpip", "5555")
            time.sleep(2)
            ok = "connected" in adbt("connect", addr).lower()
            if ok:
                self.cfg["ip"] = addr
                self.save()
                self.root.after(0, lambda: self.v["addr"].set(addr))
            self.say(f"Kablosuz bağlandı ({addr}). Kabloyu çıkarabilirsin." if ok
                     else "Bağlanılamadı. Aynı Wi-Fi'de misiniz?", ok)
        self.bg(w)

    def pair(self):
        addr, code = self.v["pair"].get().strip(), self.v["code"].get().strip()
        if not addr or not code:
            return self.say("Eşleştirme IP:port ve kodu yaz.", False)

        def w():
            r = adbt("pair", addr, code)
            ok = "successfully" in r.lower()
            self.say("Eşleşti. Şimdi alttaki IP:port ile Bağlan'a bas." if ok else r.strip(), ok)
        self.bg(w)

    def connect(self):
        addr = self.v["addr"].get().strip()
        if not addr:
            return self.say("IP:port yaz.", False)

        def w():
            r = adbt("connect", addr)
            ok = "connected to" in r or "already connected" in r
            if ok:
                self.cfg["ip"] = addr
                self.save()
            self.say("Bağlandı: " + addr if ok else r.strip(), ok)
        self.bg(w)

    def keyev(self, k):
        self.bg(lambda: adbt("shell", "input", "keyevent", str(KEYEV[k])))

    def snap(self):
        def w():
            d = adb("exec-out", "screencap", "-p")
            if not d.startswith(b"\x89PNG"):
                return self.say("Ekran görüntüsü alınamadı.", False)
            folder = os.path.join(os.path.expanduser("~"), "Desktop")
            if not os.path.isdir(folder):
                folder = os.path.expanduser("~")
            path = os.path.join(folder, time.strftime("telefon_%Y%m%d_%H%M%S.png"))
            with open(path, "wb") as f:
                f.write(d)
            self.say("Kaydedildi: " + path)
        self.bg(w)

    # ----- yansıtma -----
    def toggle_mirror(self):
        if self.mirror and self.mirror.poll() is None:
            self.mirror.terminate()
            self.mbtn.config(text="▶  Yansıtmayı başlat")
            return
        cmd = [SCRCPY, "--video-bit-rate", f"{self.v['bitrate'].get()}M"]
        if self.v["size"].get() != "0":
            cmd += ["--max-size", self.v["size"].get()]
        for k, flag in (("top", "--always-on-top"), ("awake", "--stay-awake"),
                        ("off", "--turn-screen-off"), ("nokey", "--keyboard=disabled")):
            if self.v[k].get():
                cmd.append(flag)
        try:
            self.mirror = subprocess.Popen(cmd, creationflags=NOWIN)
            self.mbtn.config(text="■  Yansıtmayı durdur")
        except FileNotFoundError:
            messagebox.showerror("scrcpy yok", "scrcpy bulunamadı. PhoneLink-Baslat.bat ile aç.")

    # ----- tuş eşleme -----
    def toggle_keys(self):
        if not self.keys_on:
            if keyboard is None:
                return messagebox.showerror("pynput yok", "Önce çalıştır: pip install pynput")
            self.shell.start()
            self.listener = keyboard.Listener(on_press=self.on_press, on_release=self.on_release)
            self.listener.start()
            self.keys_on = True
            self.kbtn.config(text="Tuş eşleme: AÇIK")
            self.kbtn.style(True)
        else:
            if self.listener:
                self.listener.stop()
            self.shell.stop()
            self.keys_on = False
            self.kbtn.config(text="Tuş eşleme: KAPALI")
            self.kbtn.style(False)

    def keyname(self, key):
        return norm(getattr(key, "char", None) or getattr(key, "name", key))

    def on_press(self, key):
        n = self.keyname(key)
        if n in self.down:
            return
        self.down.add(n)
        m = next((m for m in self.maps if m["key"] == n), None)
        if not m:
            return
        a, x, y = m["action"], m["x"], m["y"]
        if a in KEYEV:
            self.shell.send(f"input keyevent {KEYEV[a]}")
        elif a == "tap":
            self.shell.send(f"input tap {x} {y}")
        else:
            dx, dy = SWIPE[a]
            self.shell.send(f"input swipe {x} {y} {max(0, x + dx)} {max(0, y + dy)} 150")

    def on_release(self, key):
        self.down.discard(self.keyname(key))

    # ----- kayıt -----
    def load(self):
        try:
            with open(CFG) as f:
                return json.load(f)
        except Exception:
            return {}

    def save(self):
        self.cfg["maps"] = self.maps
        with open(CFG, "w") as f:
            json.dump(self.cfg, f, indent=2)

    def changed(self):
        self.save()
        self.fill()

    def fill(self):
        self.tree.delete(*self.tree.get_children())
        for m in self.maps:
            xy = ("—", "—") if m["action"] in KEYEV else (m["x"], m["y"])
            self.tree.insert("", "end", values=(m["key"], m["action"], *xy))

    def delete(self):
        for it in self.tree.selection():
            key = str(self.tree.item(it)["values"][0])
            self.maps = [m for m in self.maps if str(m["key"]) != key]
        self.changed()

    def quit(self):
        if self.listener:
            self.listener.stop()
        self.shell.stop()
        if self.mirror and self.mirror.poll() is None:
            self.mirror.terminate()
        self.root.destroy()


if __name__ == "__main__":
    App().root.mainloop()
