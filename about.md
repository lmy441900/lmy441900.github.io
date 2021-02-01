---
layout: page
title: About
permalink: /about/
---

I am a...

- (Ongoing) Master of Science in Internet of Things (MSc IoT) at [University of Southampton][uos].
- Bachelor of Science in Computer Science (and Technology) (BSc CS) at [Beijing Normal University - Hong Kong Baptist University United International College (BNU-HKBU UIC)][uic].
- (Not-very-active) Developer at [Anthon Open Source Community (AOSC)][aosc].
- HAM (amateur radio operator), level A (elementary), callsign **BI7PXN**.

Blog posts are in English or Chinese, depending on my mood.

- [GitHub][github]
- [LinkedIn][linkedin]
- [CV (coming soon)][cv]

[uic]:      https://uic.edu.hk
[uos]:      https://www.southampton.ac.uk/
[aosc]:     https://aosc.io
[github]:   https://github.com/lmy441900
[linkedin]: https://www.linkedin.com/in/lmy441900/
[cv]:       #

## GPG / PGP

I have switched my PGP key to a newly generated `ed25519 / cv25519` one, and I'm applying some best practices I learned after I've created my last PGP key.

- ~~My old key: `42F6 3E9D 68B9 884B 414D  4185 1029 4E7C 4008 E282`, has been revoked.~~
- My new key: `E6C7 4782 A1FB EE74 1D09  885F D274 286F 672C 800A`.

Subkey usage explanation:

- `ed25519/ABDA5B82F36D3DB2 2019-10-19 [S]`: main signing key. Located in an air-gapped machine. Not really used.
- `cv25519/1B318A5615C632D0 2019-10-19 [E]`: main encryption key. Located in an air-gapped machine. Not really used.
- `rsa2048/943D73C9AD8AFB50 2019-10-19 [S]`: On-card signing key. Located in a smart card (Yubikey 4; it's `rsa2048` because Yubikey 4 doesn't support 25519). Used to sign Git commits and emails on various devices.

Fetch my key from [keys.openpgp.org][koo] or [SKS keyservers][sks] ([not recommended][sks-death]).

If you have signed on my old key, you can trust my new key since I've signed it with my old key in prior to the revocation. (Unfortunately this process is still not a best practice!)

[koo]: https://keys.openpgp.org/
[sks]: https://sks-keyservers.net/
[sks-death]: https://code.firstlook.media/the-death-of-sks-pgp-keyservers-and-how-first-look-media-is-handling-it

## Contact

You can contact me over:

- [Telegram][tg].
- Email:
  - **junde at yhi punkt moe**: personal
  - **lmy441900 at** ...
    - **live dot com**: personal alternative
    - **aosc dot io**: work, AOSC, and development related
- Radio: [QSO][qso]: 147.950 MHz / 430.050 MHz; FM

[tg]:  https://t.me/lmy441900
[qso]: https://en.wikipedia.org/wiki/Contact_(amateur_radio)

> 希望你也可以活成自己想要的样子。
>
> ——夜岚Arashi
