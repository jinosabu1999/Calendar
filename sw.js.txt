/* Calendar — service worker (optional companion).
   Place next to index.html. Provides:
   1. Offline caching of the app (works from any path — incl. GitHub Pages subpaths).
   2. Best-effort background reminders: the page posts the upcoming reminder
      schedule; while this worker stays alive it fires due notifications.
      (Browsers may idle-out workers — for guaranteed delivery a push server
      would be required. In-app reminders keep working regardless.) */
const CACHE = 'calendar-v1';            // bump to purge previously cached versions
const OLD_CACHES = ['calendara-v1'];    // legacy name from before the rename
const ASSETS = ['./', './index.html'];
let schedule = [];

self.addEventListener('install', function (e) {
    e.waitUntil(
        caches.open(CACHE)
            .then(function (c) { return c.addAll(ASSETS); })
            .then(function () { return self.skipWaiting(); })
    );
});

self.addEventListener('activate', function (e) {
    e.waitUntil(
        caches.keys().then(function (keys) {
            return Promise.all(
                keys.filter(function (k) { return k !== CACHE && OLD_CACHES.indexOf(k) === -1; })
                    .map(function (k) { return caches.delete(k); })
            );
        }).then(function () { return self.clients.claim(); })
    );
});

self.addEventListener('fetch', function (e) {
    if (e.request.method !== 'GET') return;
    // network-first so app updates land on refresh; cache fallback when offline
    e.respondWith(
        fetch(e.request).then(function (res) {
            try {
                const clone = res.clone();
                caches.open(CACHE).then(function (c) { c.put(e.request, clone); });
            } catch (err) { /* noop */ }
            return res;
        }).catch(function () {
            return caches.match(e.request).then(function (hit) {
                return hit || caches.match('./index.html');
            });
        })
    );
});

self.addEventListener('message', function (e) {
    const msg = e.data || {};
    if (msg.type === 'schedule') {
        schedule = msg.items || [];
        checkDue();
    }
    if (msg.type === 'clear') schedule = [];
});

function checkDue() {
    const now = Date.now();
    schedule = schedule.filter(function (it) {
        if (it.at <= now && now <= it.at + 15 * 60000) {
            try { self.registration.showNotification(it.title, { body: it.body, tag: it.key }); } catch (err) { /* noop */ }
            return false;
        }
        return it.at > now; // drop long-overdue
    });
}

let timer = null;
function loop() {
    checkDue();
    timer = setTimeout(loop, 25000);
}
loop();

self.addEventListener('notificationclick', function (e) {
    e.notification.close();
    e.waitUntil(
        self.clients.matchAll({ type: 'window', includeUncontrolled: true }).then(function (list) {
            for (let i = 0; i < list.length; i++) { const c = list[i]; if ('focus' in c) return c.focus(); }
            return self.clients.openWindow('./');
        })
    );
});
