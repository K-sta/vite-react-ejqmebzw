# vite-react

[Edit on StackBlitz ⚡️](https://stackblitz.com/edit/vite-react)import React, { useEffect, useState, useRef } from "react";

// ================================
// 📅 期限管理アプリ（StackBlitz対応版）
// ================================
export default function App() {
  const [tasks, setTasks] = useState(() => {
    try {
      const raw = localStorage.getItem("deadlines:v1");
      return raw ? JSON.parse(raw) : [];
    } catch {
      return [];
    }
  });
  const [title, setTitle] = useState("");
  const [due, setDue] = useState("");
  const [remindAt, setRemindAt] = useState("");
  const [repeat, setRepeat] = useState("none");
  const [editingId, setEditingId] = useState(null);
  const [query, setQuery] = useState("");
  const [showOnlyDueSoon, setShowOnlyDueSoon] = useState(false);
  const audioRef = useRef(null);

  useEffect(() => {
    localStorage.setItem("deadlines:v1", JSON.stringify(tasks));
  }, [tasks]);

  useEffect(() => {
    const id = setInterval(checkReminders, 30 * 1000);
    checkReminders();
    return () => clearInterval(id);
  }, [tasks]);

  function checkReminders() {
    const now = new Date();
    tasks.forEach((t) => {
      if (t.reminded || !t.remindAt) return;
      const r = new Date(t.remindAt);
      if (r <= now) {
        triggerReminder(t);
        if (t.repeat === "none") {
          setTasks((prev) =>
            prev.map((p) => (p.id === t.id ? { ...p, reminded: true } : p))
          );
        } else {
          const next = computeNextReminder(t.remindAt, t.repeat);
          if (next) {
            setTasks((prev) =>
              prev.map((p) => (p.id === t.id ? { ...p, remindAt: next } : p))
            );
          }
        }
      }
    });
  }

  function computeNextReminder(remindAtIso, repeat) {
    const d = new Date(remindAtIso);
    if (repeat === "daily") d.setDate(d.getDate() + 1);
    else if (repeat === "weekly") d.setDate(d.getDate() + 7);
    else if (repeat === "monthly") d.setMonth(d.getMonth() + 1);
    else return null;
    return d.toISOString();
  }

  function triggerReminder(task) {
    try {
      audioRef.current?.play();
    } catch {}
    if ("Notification" in window && Notification.permission === "granted") {
      const n = new Notification(task.title || "期限リマインダー", {
        body: `期限: ${formatDateTime(task.due)}`,
        tag: `deadline-${task.id}`,
      });
      n.onclick = () => window.focus();
    } else {
      alert(`🔔 リマインダー: ${task.title}\n期限: ${formatDateTime(task.due)}`);
    }
  }

  function requestNotificationPermission() {
    if (!("Notification" in window)) {
      alert("このブラウザは通知をサポートしていません。");
      return;
    }
    Notification.requestPermission().then((perm) => {
      if (perm === "granted") alert("通知が許可されました！");
    });
  }

  function addOrUpdateTask(e) {
    e.preventDefault();
    if (!title || !due) {
      alert("タイトルと期限は必須です。");
      return;
    }
    if (editingId) {
      setTasks((prev) =>
        prev.map((t) =>
          t.id === editingId ? { ...t, title, due, remindAt, repeat } : t
        )
      );
    } else {
      const id = Math.random().toString(36).slice(2, 9);
      setTasks((prev) => [
        ...prev,
        {
          id,
          title,
          due,
          remindAt,
          repeat,
          createdAt: new Date().toISOString(),
          reminded: false,
        },
      ]);
    }
    resetForm();
  }

  function resetForm() {
    setTitle("");
    setDue("");
    setRemindAt("");
    setRepeat("none");
    setEditingId(null);
  }

  function startEdit(t) {
    setEditingId(t.id);
    setTitle(t.title);
    setDue(t.due);
    setRemindAt(t.remindAt || "");
    setRepeat(t.repeat || "none");
  }

  function removeTask(id) {
    if (!confirm("本当に削除しますか？")) return;
    setTasks((prev) => prev.filter((t) => t.id !== id));
  }

  function snooze(task, minutes) {
    const next = new Date(Date.now() + minutes * 60 * 1000).toISOString();
    setTasks((prev) =>
      prev.map((t) => (t.id === task.id ? { ...t, remindAt: next, reminded: false } : t))
    );
  }

  function formatDateTime(iso) {
    if (!iso) return "";
    const d = new Date(iso);
    return d.toLocaleString();
  }

  const filtered = tasks
    .filter((t) =>
      query ? t.title.toLowerCase().includes(query.toLowerCase()) : true
    )
    .filter((t) =>
      showOnlyDueSoon ? new Date(t.due) - Date.now() < 3 * 24 * 60 * 60 * 1000 : true
    )
    .sort((a, b) => new Date(a.due) - new Date(b.due));

  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-50 to-white p-4">
      <div className="max-w-4xl mx-auto">
        <header className="flex items-center justify-between mb-6">
          <h1 className="text-2xl font-extrabold">期限管理アプリ 🗓️</h1>
          <button
            onClick={requestNotificationPermission}
            className="px-3 py-1 rounded-lg border"
          >
            通知の許可
          </button>
        </header>

        <main className="grid md:grid-cols-2 gap-6">
          <form onSubmit={addOrUpdateTask} className="p-4 bg-white rounded-2xl shadow">
            <h2 className="font-semibold mb-3">タスクを追加 / 編集</h2>

            <label className="block text-sm">タイトル</label>
            <input
              value={title}
              onChange={(e) => setTitle(e.target.value)}
              className="w-full p-2 border rounded mb-2"
            />

            <label className="block text-sm">期限（必須）</label>
            <input
              value={due}
              onChange={(e) => setDue(e.target.value)}
              type="datetime-local"
              className="w-full p-2 border rounded mb-2"
            />

            <label className="block text-sm">リマインド日時（任意）</label>
            <input
              value={remindAt}
              onChange={(e) => setRemindAt(e.target.value)}
              type="datetime-local"
              className="w-full p-2 border rounded mb-2"
            />

            <label className="block text-sm">繰り返し</label>
            <select
              value={repeat}
              onChange={(e) => setRepeat(e.target.value)}
              className="w-full p-2 border rounded mb-3"
            >
              <option value="none">なし</option>
              <option value="daily">毎日</option>
              <option value="weekly">毎週</option>
              <option value="monthly">毎月</option>
            </select>

            <button type="submit" className="px-4 py-2 bg-indigo-600 text-white rounded">
              {editingId ? "更新" : "追加"}
            </button>
          </form>

          <section className="p-4 bg-white rounded-2xl shadow">
            <div className="flex items-center justify-between mb-3">
              <h2 className="font-semibold">タスク一覧</h2>
              <div className="flex gap-2">
                <input
                  placeholder="検索..."
                  value={query}
                  onChange={(e) => setQuery(e.target.value)}
                  className="p-2 border rounded"
                />
                <label className="flex items-center gap-2 text-sm">
                  <input
                    type="checkbox"
                    checked={showOnlyDueSoon}
                    onChange={(e) => setShowOnlyDueSoon(e.target.checked)}
                  />
                  3日以内
                </label>
              </div>
            </div>

            {filtered.length === 0 ? (
              <p className="text-sm text-slate-500">タスクがありません。</p>
            ) : (
              <ul className="space-y-3">
                {filtered.map((t) => (
                  <li
                    key={t.id}
                    className="border rounded p-3 flex flex-col md:flex-row md:items-center md:justify-between gap-2"
                  >
                    <div>
                      <div className="font-medium">{t.title}</div>
                      <div className="text-sm text-slate-600">
                        期限: {formatDateTime(t.due)}
                      </div>
                      <div className="text-sm text-slate-500">
                        リマインド:{" "}
                        {t.remindAt ? formatDateTime(t.remindAt) : "未設定"}{" "}
                        {t.repeat !== "none" ? `(${t.repeat})` : ""}
                      </div>
                    </div>
                    <div className="flex gap-2 flex-wrap">
                      <button
                        onClick={() => startEdit(t)}
                        className="px-3 py-1 border rounded"
                      >
                        編集
                      </button>
                      <button
                        onClick={() => removeTask(t.id)}
                        className="px-3 py-1 border rounded"
                      >
                        削除
                      </button>
                      <button
                        onClick={() => snooze(t, 5)}
                        className="px-3 py-1 border rounded"
                      >
                        スヌーズ5分
                      </button>
                    </div>
                  </li>
                ))}
              </ul>
            )}
          </section>
        </main>

        <footer className="mt-6 text-sm text-slate-500">
          ※ ローカルに保存されます。通知を受けるには「通知の許可」を押してください。
        </footer>

        <audio ref={audioRef} src={createBeepDataURI()} />
      </div>
    </div>
  );
}

// 短い beep 音
function createBeepDataURI() {
  return "data:audio/wav;base64,UklGRiQAAABXQVZFZm10IBAAAAABAAEAESsAACJWAAACABAAZGF0YQAAAAA=";
}