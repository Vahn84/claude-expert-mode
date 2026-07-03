# Round 2 — Cold control: avatar-upload endpoint (no interview priming)

Session `9e295693-c7d9-4db6-9ffb-be6c08324294`. Verbatim text turns; tool traffic omitted.


---

## Turn 1 — Prompt (cold)

Quick one: can you write me an Express endpoint that lets a logged-in user upload a profile avatar image, stores it, and returns the URL? Just the handler, Node/Express, however you'd normally write it. Keep it short.


---

## Turn 2 — Opus 4.8 (cold)

Here's a compact but production-sane handler using `multer`. The key things I bake in even for a "short" version: don't trust the client filename, restrict to images, and cap the size.

```js
const express = require('express');
const multer = require('multer');
const crypto = require('crypto');
const path = require('path');

const router = express.Router();

// Only allow real image types, and generate our own filename — never trust originalname.
const ALLOWED = { 'image/jpeg': '.jpg', 'image/png': '.png', 'image/webp': '.webp' };

const upload = multer({
  storage: multer.diskStorage({
    destination: 'uploads/avatars',
    filename: (req, file, cb) => {
      const ext = ALLOWED[file.mimetype];
      cb(null, `${req.user.id}-${crypto.randomBytes(8).toString('hex')}${ext}`);
    },
  }),
  limits: { fileSize: 2 * 1024 * 1024 }, // 2 MB
  fileFilter: (req, file, cb) => {
    if (!ALLOWED[file.mimetype]) return cb(new Error('Only JPEG, PNG or WebP images are allowed'));
    cb(null, true);
  },
});

// requireAuth populates req.user; upload.single parses the "avatar" field.
router.post('/me/avatar', requireAuth, upload.single('avatar'), async (req, res, next) => {
  try {
    if (!req.file) return res.status(400).json({ error: 'No image uploaded' });

    const url = `${req.protocol}://${req.get('host')}/uploads/avatars/${req.file.filename}`;

    // Persist the URL on the user record.
    await User.findByIdAndUpdate(req.user.id, { avatarUrl: url });

    res.status(200).json({ url });
  } catch (err) {
    next(err);
  }
});

module.exports = router;
```

Assumes you already have `requireAuth` (sets `req.user`) and a `User` model, and that `uploads/` is served statically (`app.use('/uploads', express.static('uploads'))`).

A few things I'd flag before this goes near production — happy to fold any in:

- **Content sniffing**: `file.mimetype` comes from the client and can be spoofed. For real safety, verify magic bytes (e.g. `file-type`) or re-encode through `sharp`, which also lets you resize/strip EXIF.
- **Old avatars**: this leaves the previous file on disk on every re-upload — you'd want to delete the old one.
- **Storage**: local disk doesn't survive a container redeploy — for the dndlore NAS/Docker setup you'd likely want a mounted volume or object storage.

Want me to drop this into a file in the dndlore backend and wire it to your actual auth middleware / `User` model instead of the placeholders?
