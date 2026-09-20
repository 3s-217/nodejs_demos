# Github Webhook server

This demo illustrates a secure, modular, and adaptable webhook handler built using Node.js and Express. It showcases core concepts like routing, GitHub API integration, HMAC signature validation, and database mocking for lightweight testing.

## Features

- Secure webhook handling with HMAC signature validation.
- Modular routing logic (`autho`, `proxy`).
- Simple database mocking (`mockDB`) for demonstration purposes.
- Flexible endpoints suitable for production adaptation.

### Install deps

```bash
npm i express compression morgan
```

### mockDB

```js
function mockDB(data = []) {
    const rws = data;
    this.get = (o) => {
        let k = Object.keys(o);
        for (let r of rws) {
            if (k.every(v => r[v] == o[v]))
                return r;
        }
    };
}
```

### Main script

The ***json*** method is implemented directly within the ***__o*** class as a static function. You can find its definition in the [github.com/3s217/coreWeb-js](https://github.com/3s-217/coreWeb-js)
repository.

```js
const express = require("express");
const { BlockList } = require('net');
const crypto = require("node:crypto");
const app = express();
const port = 8080;
const dev = true;
const [log, err] = ['log', 'error'].flatMap(v => console[v].bind(console));
//
app.use(express.json());
//
//* Compression
//
const compression = require("compression");
app.use(compression());
//
//* Logging
//
const logger = require("morgan");
app.use(logger());
//
//* Github endpoints
//
const ghUrl = 'https://api.github.com/repos/';
const GH = {
    dispatch: (o, r) => `${ghUrl}${o}/${r}/dispatch`,
    meta: ghUrl.replace("repo/", 'meta'),
    hook: (o, r) => `${ghUrl}${o}/${r}/hooks`,
    repo: (o, r) => `${ghUrl}${o}/${r}`,
    action: (o, r) => `${ghUrl}${o}/${r}/actions/workflows`,
    release: (o, r) => `${ghUrl}${o}/${r}/releases`,
    tag: (o, r) => `${ghUrl}${o}/${r}/tags`,
    branch: (o, r) => `${ghUrl}${o}/${r}/branches`
};
const APP = {
    //* The mockDB is not for production use
    //* replace it with proper db 
    //* also you will need to update function autho's [APP.db.get] logic with your db query method
    db: new mockDB([{
        org_from: 'org_1',
        repo_from: 'repo_1',
        org_to: 'org_1',
        repo_to: 'repo_3',
        url: crypto.randomUUID(),
        secret: crypto.randomUUID(),
        pat: crypto.randomUUID(),
    }]),
    blk: !dev ? getIpList() : 0
};
async function autho(rq, rs, next) {
    const des = () => rs.status(204).send();
    if (!dev) {
        if (!APP.blk.check(rq.ip))
            return des();
    }
    if (rq.headers["content-type"] !== "application/json")
        return rs.status(415).json({ error: "Unsupported Data Type. Use application/json." });
    const repo = rq.body?.repository?.name;
    if (!repo) return des();
    //* replace this part with your db logic
    let gt = APP.db.get({ repo_from: repo, url: rq.params.x });
    if (!gt || gt?.length < 1) return des();
    gt = gt[0];
    if (!veriSig(rq, gt.secret)) return des();
    rq.rw = gt;
    next();
}
async function proxy(rq, rs) {
    let gt = rq.rw;
    let gh = dev ? "http://localhost:" + port + "/dev/" + gt.ower_to + "/" + gt.repo_to :
        GH.dispatch(gt.ower_to, gt.repo_to);
    const headers = { ...rq.headers, Authorization: `token ${gt.pat}`, },
        ct = 'content-length',
        st = i => (delete i?.headers[ct], i?.headers.forEach((v, k) => rs.setHeader(k, v)));
    [ct, 'connection', 'host'].forEach(k => delete headers[k]);
    let fwd = await fetch(gh, { method: "POST", headers, body: json(rq.body) })
        .catch(e => {
            st(e);
            (log(e), rs.status(e?.status || 500).send(e?.body || { error: e?.type || 'error' }));
        });
    if (fwd) {
        st(fwd);
        rs.status(fwd?.status).send(await fr?.text().catch(e => e) ?? '');
    }
}
function veriSig(rq, secret, dbg) {
    const sig = rq.headers['x-hub-signature-256']; // GitHub's signature header
    const pyd = json(rq.body);
    if (dbg)
        log(sig, crypto.createHmac('sha256', secret).update(pyd).digest('hex'), pyd);
    return !pyd ? pyd : sig === `sha256=${enc.c.createHmac('sha256', secret).update(pyd).digest('hex')}`;
}
async function getIpList(url = GH.meta, key = "hooks") {
    let rs = await fetch(url).catch(e => (log(e), { ok: !!0 }));
    if (!rs.ok) return false;
    rs = await rs.json();
    let bl = new BlockList();
    let a;
    for (let i of rs[key]) a = i.split("/"), bl.addSubnet(a[0], parseInt(a[1]));
    return bl;
}
//============
app.use(logger(dev ? 'dev' : "short"));
app.post('/hook/:x', autho, proxy);
//* dev route 
if (dev)
    app.post('/dev/:x/:y', (rq, rs, next) => {
        //log('proxy-test-body:\n', rq.body);
        //log('proxy-test-headers:\n', rq.headers);
        return !rq.body ||
           (rq.body && (typeof rq.body != 'object' || Array.isArray(rq.body))) ? 
            next() : rs.send(rq.body);
    });
//* Error catch all
app.use((rq, rs) => rs.status(204).send());
//
//* app listen
//
app.listen(port);

```

## Notes

- Replace `mockDB` with your preferred database solution for production use.
- Update routing logic and security measures to match your requirements.
- This skeleton is meant to be adaptable for private webhook endpoints.
