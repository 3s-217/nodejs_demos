# fsio - Dynamic File Monitoring Class

## Overview

`fsio` is a lightweight and efficient class designed to monitor and manage interactions with a single file in real-time. It simplifies file tracking by handling changes, renames, and event-based notifications, making it a practical solution for scenarios like log monitoring, file synchronization, or data persistence.

## Key Features

- **Event-Driven Monitoring**: Tracks file changes (`change`) and renames (`rename`), emitting detailed events for external use.
- **Daily Reset Logic**: Automatically resets counters and state at the start of a new day.
- **Persistent Data Storage**: Maintains file statistics and history in `fsio-data.json`.
- **Line-Level Tracking**: Reads the number of lines in the monitored file and tracks changes efficiently.
- **Encapsulation**: Uses modern JavaScript private fields (`#`) to protect internal logic.

---

## Real-World Applications

- Real-time log file monitoring.
- Automated data processing workflows.
- Persistent tracking for auditing systems.

## Class fsio

### Full Class Definition

***The json method is implemented directly within the __o class as a static function. You can find its definition in the [github.com/3s217/coreWeb-js](https://github.com/3s-217/coreWeb-js)
repository.***

```javascript
const { writeFileSync: wfs, readFileSync: rfs, existsSync, 
watch, stat, createReadStream: crs, statSync } = require('fs');
const { createInterface: cI } = require("readline");
const events = require("events");
const { once } = events;
const path = require("path");
const [log, err] = ['log', 'error'].flatMap(v => console[v].bind(console));
const preZ = n => (n <= 9 ? "0" + n : n);
//
class fsio extends events {
    #len = 0;
    get len() { return this.#len; }
    #cf = { fl: '', rt: "" };
    #ti;// used for debounce timer 
    #tday;// today's date for persistence
    get date() { return this.#tday; }
    get cf() { return this.#cf; }
    // change this part to your server's time zone 
    get day() { var td = new Date(); return `${td.getFullYear()}${preZ(td.getUTCMonth())}${preZ(td.getUTCDate())}`; };
    constructor(o, tt = 1 * 1e3) {
        super();
        let t = this;
        t.tm = tt;
        t.tday = t.day;
        t.#cf.rt = o.rt || o.appDataDir;
        t.#cf.fl = path.join(t.#cf.rt, 'fsio-data.json');
        let dir, ex, tdo, pt = path.parse(tdo = (o.fl || o.file));
        dir = existsSync(tdo) && statSync(tdo);
        if (!dir || (dir && !(ex = dir.isFile())))
            return err(!ex ? "Its not a file" : "File missing");
        t.nm = (pt.name + pt.ext);
        t.dir = pt.dir.replaceAll(path.sep, '/');
        t.path = t.dir + "/" + t.nm;
        log("Path-type:", pt = ex ? "File" : "Dir",
            `\n${pt}-Name:`, t.dir + "/" + t.nm);
        if (ex)
            log(`Syncing: ${t.nm}`),
                t.#per();
        t.wth = t.#watch.bind(t);
        t.watch = watch(t.dir, t.wth);
        log(`Watching: ${t.dir}\n${pt}-Name: ${t.nm} `);
    }
    #nDay() {
        let t = this, td = t.day;
        if (t.tday != td) {
            t.tday = td; t.#len = 0; t.sz = 0;
            stat(path.join(t.dir, t.nm), (e, d) => {
                t.#persist(d.size > 10000 && {});
                log('new Day');
            });
        }
    }
    #per() {
        let t = this, n = t.nm, z = t.cf.fl,
            f = () => wfs(z, json({ [t.day]: { [t.dir]: { [n]: t.len } } }));
        if (existsSync(z)) {
            var a = json(rfs(z, { encoding: "utf8" })), b;
            a ? (b = a[t.tday]?.[t.dir]) && b[n] > -1 &&
                (t.#len = b[n])
                : f();
        }
        else f();
        this.lines(t.nm, t.len).then(v => (t.len = v.total, t.emit('info', v)));
    }
    #act = [];
    #watch(ev, file) {
        let t = this;
        if (ev == 'rename')
            t.#act.push({ ev, file });
        else if (ev == 'change') {
            let f = t.#act[t.#act.length - 1];
            (f && f.file == file && f.ev == ev) ? 0 :
                t.#act.push({ ev, file });
        }
        t.#ti && (clearTimeout(t.#ti), t.#ti = null);
        t.#ti = setTimeout(() => t.#action(), t.tm);
    }
    async #action() {
        const { nm, dir, len } = this, at = this.#act;
        let c = at.shift(), d;
        if (!c) return;
        if (c.ev == 'rename') {
            if (at[0]?.ev == 'rename') {
                if (c.file == nm || at[0].file == nm) {
                    log("Renamed: from:", c.file, 'to:', at[0].file);
                    this.emit("info", { event: 'rename', old: c.file, new: at[0].file });
                    c = at.shift();
                    at[0]?.ev == "change" && at[0].file == c.file &&
                        at.shift();
                }
                else at.splice(0, at[1]?.ev == "change" &&
                    at[0].file == c.file ? 2 : 1);
            }
            else if (c.file == nm) {
                if (d = existsSync(path.join(dir, c.file)))
                    log("Created:", c.file), this.#nDay();
                else log("Deleted:", c.file);
                this.emit("info", { event: d ? 'created' : 'deleted', file: c.file });
            }
        }
        else if (c.ev == 'change') {
            if (c.file == nm && existsSync(path.join(dir, c.file))) {
                log("File-change:", c.file);
                let e = await this.lines(c.file, len);
                this.#len = e.total;
                this.#persist();
                this.emit('change', e);
            }
        }
    }
    async #persist(o) {
        var t = this, a = o || json(rfs(t.cf.fl, { encoding: "utf8" }));
        if (!a) return;
        !a[t.tday] && (a[t.tday] = { [t.dir]: {} });
        a[t.tday][t.dir][t.nm] = t.len;
        wfs(t.cf.fl, json(a));
    }
    async lines(file, last = 0) {
        let t = this, fl = path.join(t.dir, file);
        let rl, ln = [], ct = 0;
        try {
            rl = cI({ input: crs(fl), crlfDelay: Infinity, terminal: !!0 });
            rl.on('line', l => (ln.push(l), ct += 1));
            await once(rl, 'close');
        }
        catch (e) { err(e); }
        if (ct > 0 && ct > last)
            ln.splice(0, ln.length - (ct - last));
        return { total: ct, file, ln, dif: ln.length };
    }
}
```

## Code Snippet

Below is an excerpt showcasing how `fsio` can be used:

```javascript
    const watchLog = new fsio({
        appDataDir: __dirname,
        file: path.join(__dirname, "access_log"),
    });
    watchLog.on("change", evt => log('change', evt))
        .on("info", evt => log('info', evt));
```

Why Choose fsio?
fsio provides an elegant solution for log management, combining real-time file monitoring with modular integration, perfect for modern backend systems.
