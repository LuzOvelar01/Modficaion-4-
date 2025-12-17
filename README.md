"""
Task Manager - Gestor de Tareas Personales
Python 3.x + Tkinter (ttk) + SQLite
"""

import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
from datetime import datetime

DB_FILE = "tasks.db"

# ---------- Util ----------
def validar_fecha(fecha_texto):
    try:
        datetime.strptime(fecha_texto, "%Y-%m-%d")
        return True
    except:
        return False

# ---------- DB ----------
class TaskDB:
    def __init__(self, db_path=DB_FILE):
        self.conn = sqlite3.connect(db_path)
        self.create_table()

    def create_table(self):
        c = self.conn.cursor()
        c.execute("""
        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            description TEXT,
            due_date TEXT,
            priority TEXT,
            status TEXT,
            created_at TEXT
        )""")
        self.conn.commit()

    def add_task(self, title, description, due, priority, status):
        c = self.conn.cursor()
        c.execute("""
        INSERT INTO tasks (title, description, due_date, priority, status, created_at)
        VALUES (?, ?, ?, ?, ?, ?)""",
        (title, description, due, priority, status, datetime.now().isoformat()))
        self.conn.commit()

    def update_task(self, task_id, title, description, due, priority, status):
        c = self.conn.cursor()
        c.execute("""
        UPDATE tasks SET title=?, description=?, due_date=?, priority=?, status=? WHERE id=?""",
        (title, description, due, priority, status, task_id))
        self.conn.commit()

    def delete_task(self, task_id):
        c = self.conn.cursor()
        c.execute("DELETE FROM tasks WHERE id=?", (task_id,))
        self.conn.commit()

    def get_all_tasks(self):
        c = self.conn.cursor()
        c.execute("SELECT id, title, description, created_at, due_date, priority, status FROM tasks ORDER BY id DESC")
        return c.fetchall()

    def get_task(self, task_id):
        c = self.conn.cursor()
        c.execute("SELECT id, title, description, created_at, due_date, priority, status FROM tasks WHERE id=?", (task_id,))
        return c.fetchone()

    def search_tasks(self, q):
        c = self.conn.cursor()
        q = f"%{q}%"
        c.execute("""
        SELECT id, title, description, created_at, due_date, priority, status
        FROM tasks WHERE title LIKE ? OR description LIKE ? ORDER BY id DESC""", (q, q))
        return c.fetchall()

    def close(self):
        self.conn.close()

# ---------- Formulario Agregar/Editar ----------
class TaskForm(tk.Toplevel):
    def __init__(self, master, db, on_save, task_id=None):
        super().__init__(master)
        self.title("Agregar Tarea" if not task_id else "Editar Tarea")
        self.resizable(False, False)
        self.db, self.on_save, self.task_id = db, on_save, task_id

        pad = 8
        self.columnconfigure(1, weight=1)
        ttk.Label(self, text="Título:*").grid(row=0, column=0, padx=pad, pady=pad, sticky="e")
        self.title_var = tk.StringVar()
        ttk.Entry(self, textvariable=self.title_var, width=50).grid(row=0, column=1, padx=pad, pady=pad)

        ttk.Label(self, text="Descripción:").grid(row=1, column=0, padx=pad, pady=pad, sticky="ne")
        self.desc_text = tk.Text(self, height=5, width=50)
        self.desc_text.grid(row=1, column=1, padx=pad, pady=pad)

        ttk.Label(self, text="Fecha vencimiento (YYYY-MM-DD):").grid(row=2, column=0, padx=pad, pady=pad, sticky="e")
        self.due_var = tk.StringVar()
        ttk.Entry(self, textvariable=self.due_var).grid(row=2, column=1, padx=pad, pady=pad, sticky="we")

        ttk.Label(self, text="Prioridad:").grid(row=3, column=0, padx=pad, pady=pad, sticky="e")
        self.priority_var = tk.StringVar()
        cb1 = ttk.Combobox(self, textvariable=self.priority_var, values=["Baja", "Media", "Alta"], state="readonly")
        cb1.grid(row=3, column=1, padx=pad, pady=pad); cb1.current(1)

        ttk.Label(self, text="Estado:").grid(row=4, column=0, padx=pad, pady=pad, sticky="e")
        self.status_var = tk.StringVar()
        cb2 = ttk.Combobox(self, textvariable=self.status_var, values=["Pendiente", "En progreso", "Completada"], state="readonly")
        cb2.grid(row=4, column=1, padx=pad, pady=pad); cb2.current(0)

        # Fecha creación (solo lectura, visible al editar)
        self.created_label = ttk.Label(self, text="")
        self.created_label.grid(row=5, column=1, padx=pad, pady=(4,0), sticky="w")

        frame = ttk.Frame(self); frame.grid(row=6, column=0, columnspan=2, pady=(6,pad))
        ttk.Button(frame, text="Guardar", command=self.guardar).pack(side="left", padx=5)
        ttk.Button(frame, text="Cancelar", command=self.destroy).pack(side="left")

        if self.task_id: 
            self.load_task()

    def load_task(self):
        r = self.db.get_task(self.task_id)
        if r:
            tid, title, desc, created_at, due, pr, st = r
            self.title_var.set(title)
            self.desc_text.delete("1.0", tk.END)
            self.desc_text.insert("1.0", desc or "")
            self.due_var.set(due or "")
            if pr: self.priority_var.set(pr)
            if st: self.status_var.set(st)
            # mostrar fecha de creación legible
            try:
                dt = datetime.fromisoformat(created_at)
                created_text = f"Fecha creación: {dt.strftime('%Y-%m-%d %H:%M')}"
            except:
                created_text = f"Fecha creación: {created_at or ''}"
            self.created_label.config(text=created_text)

    def guardar(self):
        if not self.title_var.get().strip():
            return messagebox.showwarning("Validación", "El título es obligatorio.")
        due = self.due_var.get().strip()
        if due and not validar_fecha(due):
            return messagebox.showwarning("Validación", "Fecha inválida (YYYY-MM-DD).")
        data = (
            self.title_var.get().strip(),
            self.desc_text.get("1.0", tk.END).strip(),
            due or None,
            self.priority_var.get(),
            self.status_var.get()
        )
        if self.task_id:
            self.db.update_task(self.task_id, *data)
        else:
            self.db.add_task(*data)
        self.on_save(); self.destroy()

# ---------- Ventana Acerca ----------
class AboutWindow(tk.Toplevel):
    def __init__(self, master, db):
        super().__init__(master)
        self.title("Acerca / Estadísticas")
        self.resizable(False, False)
        ttk.Label(self, text="Gestor de Tareas Personales", font=("TkDefaultFont", 12, "bold")).pack(pady=8)
        tasks = db.get_all_tasks()
        total = len(tasks)
        comp = sum(1 for t in tasks if t[6] == "Completada")
        ttk.Label(self, text=f"Total de tareas: {total}").pack(anchor="w", padx=12)
        ttk.Label(self, text=f"Tareas completadas: {comp}").pack(anchor="w", padx=12)
        ttk.Label(self, text=f"Tareas pendientes: {total - comp}").pack(anchor="w", padx=12)
        ttk.Button(self, text="Cerrar", command=self.destroy).pack(pady=8)

# ---------- App Principal ----------
class TaskManagerApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("Gestor de Tareas Personales")
        self.geometry("950x520")
        self.db = TaskDB()
        self.style = ttk.Style(self)
        try: self.style.theme_use("clam")
        except: pass
        self.create_widgets()
        self.load_tasks()

    def create_widgets(self):
        top = ttk.Frame(self); top.pack(fill="x", padx=8, pady=8)
        ttk.Label(top, text="Buscar:").pack(side="left")
        self.search_var = tk.StringVar()
        e = ttk.Entry(top, textvariable=self.search_var, width=30); e.pack(side="left", padx=5)
        e.bind("<Return>", lambda ev: self.search_tasks())
        ttk.Button(top, text="Ir", command=self.search_tasks).pack(side="left")
        ttk.Button(top, text="Mostrar todo", command=self.load_tasks).pack(side="left", padx=5)
        ttk.Label(top, text=" ").pack(side="left", expand=True)
        ttk.Button(top, text="Agregar", command=self.open_add).pack(side="right")
        ttk.Button(top, text="Editar", command=self.open_edit).pack(side="right", padx=4)
        ttk.Button(top, text="Eliminar", command=self.delete_selected).pack(side="right", padx=4)
        ttk.Button(top, text="Marcar Completada", command=self.mark_completed).pack(side="right", padx=4)
        ttk.Button(top, text="Acerca/Stats", command=self.open_about).pack(side="right", padx=4)

        cols = ("id","title","created","due","priority","status")
        self.tree = ttk.Treeview(self, columns=cols, show="headings")
        headers = ["ID","Título","Fecha creación","Vencimiento","Prioridad","Estado"]
        for c,h,w in zip(cols, headers, [50,320,150,120,80,100]):
            self.tree.heading(c, text=h)
            self.tree.column(c, width=w, anchor="center" if c in ("id","created","due","priority","status") else "w")
        self.tree.pack(fill="both", expand=True, padx=8, pady=(0,8))

        # Tags colores
        self.tree.tag_configure("completada", background="#C8E6C9")
        self.tree.tag_configure("alta", background="#FFCDD2")
        self.tree.tag_configure("media", background="#FFF9C4")

        bottom = ttk.Frame(self); bottom.pack(fill="x", padx=8, pady=5)
        self.info_label = ttk.Label(bottom, text="Seleccione una tarea para ver detalles.")
        self.info_label.pack(side="left")
        self.tree.bind("<Double-1>", lambda e: self.open_edit())

    def load_tasks(self):
        for r in self.tree.get_children(): self.tree.delete(r)
        rows = self.db.get_all_tasks()
        for r in rows:
            tid, title, desc, created_at, due, pr, st = r
            tag = "completada" if st=="Completada" else ("alta" if pr=="Alta" else ("media" if pr=="Media" else ""))
            # formatear created_at a algo legible (solo fecha)
            try:
                created_fmt = datetime.fromisoformat(created_at).strftime("%Y-%m-%d %H:%M")
            except:
                created_fmt = created_at or ""
            self.tree.insert("", tk.END, values=(tid, title, created_fmt, due or "", pr or "", st or ""), tags=(tag,))
        self.info_label.config(text=f"Tareas cargadas: {len(rows)}")

    def search_tasks(self):
        q = self.search_var.get().strip()
        for r in self.tree.get_children(): self.tree.delete(r)
        if not q:
            return self.load_tasks()
        rows = self.db.search_tasks(q)
        for r in rows:
            tid, title, desc, created_at, due, pr, st = r
            tag = "completada" if st=="Completada" else ("alta" if pr=="Alta" else ("media" if pr=="Media" else ""))
            try:
                created_fmt = datetime.fromisoformat(created_at).strftime("%Y-%m-%d %H:%M")
            except:
                created_fmt = created_at or ""
            self.tree.insert("", tk.END, values=(tid, title, created_fmt, due or "", pr or "", st or ""), tags=(tag,))
        self.info_label.config(text=f"Resultados: {len(rows)}")

    def get_selected_task_id(self):
        sel = self.tree.selection()
        if not sel:
            messagebox.showwarning("Atención", "Seleccione una tarea.")
            return None
        return self.tree.item(sel[0])["values"][0]

    def open_add(self):
        TaskForm(self, self.db, on_save=self.load_tasks)

    def open_edit(self):
        tid = self.get_selected_task_id()
        if tid:
            TaskForm(self, self.db, on_save=self.load_tasks, task_id=tid)

    def delete_selected(self):
        tid = self.get_selected_task_id()
        if tid and messagebox.askyesno("Confirmar", "¿Eliminar tarea?"):
            self.db.delete_task(tid)
            self.load_tasks()

    def mark_completed(self):
        tid = self.get_selected_task_id()
        if not tid: return
        r = self.db.get_task(tid)
        if r:
            # r: id, title, description, created_at, due_date, priority, status
            self.db.update_task(tid, r[1], r[2], r[4], r[5], "Completada")
            self.load_tasks()

    def open_about(self):
        AboutWindow(self, self.db)

    def on_close(self):
        self.db.close()
        self.destroy()

# ---------- Pantalla de Bienvenida ----------
def mostrar_bienvenida():
    def ingresar():
        splash.destroy()
        abrir_gestor()

    splash = tk.Tk()
    splash.title("Bienvenido")
    splash.geometry("500x300")
    splash.configure(bg="#8A2BE2")
    splash.resizable(False, False)

    splash.update_idletasks()
    x = (splash.winfo_screenwidth() // 2) - (500 // 2)
    y = (splash.winfo_screenheight() // 2) - (300 // 2)
    splash.geometry(f"500x300+{x}+{y}")

    titulo = tk.Label(splash, text="Task Manager - Gestor de Tareas Personales",
                      bg="#8A2BE2", fg="white", font=("Helvetica", 16, "bold"))
    titulo.pack(pady=(20,10))

    mensaje = tk.Label(
        splash,
        text="¡Hola! Bienvenido al Gestor de Tareas\n\nPequeños avances diarios generan grandes resultados.",
        bg="#8A2BE2", fg="white", font=("Helvetica", 14), justify="center", wraplength=400
    )
    mensaje.pack(expand=True)

    ttk.Button(splash, text="Ingresar", command=ingresar).pack(pady=20)
    splash.mainloop()

def abrir_gestor():
    app = TaskManagerApp()
    app.protocol("WM_DELETE_WINDOW", app.on_close)
    app.mainloop()

if __name__ == "__main__":
    mostrar_bienvenida()
