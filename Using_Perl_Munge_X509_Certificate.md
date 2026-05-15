# Using Perl to munge an X.509 certificate

I wanted to do something fairly quick with [X.509
certificates](http://www.ietf.org/rfc/rfc2459.txt "X.509 certificates"). My
scribbled requirements were: connect to a server and grab the public
certificate and then expose the X.509 fields programmatically. As an old Perl
hacker-at-heart I headed over to [CPAN](http://search.cpan.org "CPAN") and
grabbed a copy of [Net::SSLeay](http://search.cpan.org/~mikem/Net-SSLeay-1.49/lib/Net/SSLeay "Net::SSLeay"). It seems to be the de facto module
for this work and is recommended by [O'Reilly's excellent
book](http://shop.oreilly.com/product/9780596002701.do "O'Reilly's excellent
book") on OpenSSL that I happened to be reading. Furthermore there was a yum
installer for CentOs. Happy days!

After a bit of fiddling I'd hacked the test ssl client that comes with the
bundle to do the job. There were a couple of gotchas. Firstly I was getting
some core dumps. I never really got to the bottom of these. I thought this
might be that the yum installer didn't do a good job (because later installs
required the openssl-dev package, but yum didn't complain). Anyway I got
around the core dumps by removing the nested calls from my code. Perhaps those
warnings about [OpenSSL](http://www.openssl.org "OpenSSL") not playing nicely
with threads have some weight after all? Anyway, by this time my code was
starting to get a bit messy and I had hit another problem.

This isn't to do with Net::SSLeay directly but the fact that
[Socket](http://perldoc.perl.org/5.8.9/Socket.html "Socket") requires you to
run as root. I can see this made sense at some earlier point in history when
_being root_ was a Big Thing. But now everyone has a couple of VMs kicking
around and what's the point in making user accounts? It was a bit annoying
because I actually couldn't run as root in my target environment. Seems a lot
of kafuffle when all I wanted to do was make a standard https connection to
places like <https://www.google.com>. Not so extra-ordinary! The only work
around I've found so far is to pipe through `openssl s_client` and this works
fine, if a bit 1970s. Please comment with any better suggestions.

The next bit was to start grappling with the certificate themselves. I did
consider [Sam Vilain's OO Net::SSLeay](http://search.cpan.org/~samv/Net-SSLeay-OO-0.02/lib/Net/SSLeay/OO/X509.pm "Sam Vilain's OO Net::SSLeay") as it
looks like an improved interface and this was my main gripe with Net::SSLeay.
(I should say that I got a nice reply from the author of Net::SSLeay). But I
was worried that it was still Net::SSLeay underneath and by then Dan Sully's
[Crypt::OpenSSL::X509](http://search.cpan.org/~daniel/Crypt-OpenSSL-X509-1.800.2/X509.pm "Crypt::OpenSSL::X509") had caught my eye. It's a
really nice API. Everything just seems to be where you'd expect it. So I got
stuck in and all was well for a while. Turns out this module has problems too.
Mainly it stops dead on certificates that it doesn't understand. For example
[https://google.co.in/](https://google.co.in) has a stonker of a certificate
with several hundred X509 v3 extensions. Dan's module just fails to cope, no
warning and no nice reply from Dan. The other problem is that it isn't
finished.

![Drawing of a multi-headed Hydra](http://lh3.ggpht.com/_QW_USKRkIzk/TRz-GmQci5I/AAAAAAAAyDs/aJq_4NOSIaM/image_thumb.png).

At this point I was thinking about starting again in Java ..

The other problem with Dan's module is that the ASN.1 notation that underpins
the X.509 standard is a horrible multi-layered thing. When Dan's module get's
past the first layer it starts throwing-up gobble-de-gook. You see, not
everything is a string in the world of X.509. I mean this:  
`  
X509v3 Subject Key Identifier  
53:32:D1:B3:CF:7F:FA:E0:F1:A0:5D:85:4E:92:D2:9E:45:1D:B4:4F  
`  
became that.  
`  
X509v3 Subject Key Identifier  
..S2........].N...E..O  
`  
There is an X.509 module as part of
[Crypt::SSLeay](http://search.cpan.org/~nanis/Crypt-SSLeay-0.64/
"Crypt::SSLeay") but it's deprecated and the module is only maintained for
protocol support of the amazing [LWP](http://search.cpan.org/~gaas/libwww-perl-6.04/lib/LWP.pm "LWP") (it puts the s in https). This is a shame because
had I been able to grab the certificate from an LWP session then two birds
might have been left to [die](http://perldoc.perl.org/functions/die.html
"Perl's die command"). I also found this
[sslclient](http://www.superscript.com/ucspi-ssl/ "sslclient") which looked
perfect. But it failed the install.

Right now I'm working with
[Crypt::X509](http://search.cpan.org/~ajung/Crypt-X509-0.51/lib/Crypt/X509.pm
"Crypt::X509") by Alexander Jung. It seems to be a bit obsessed with
[LDAP](http://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol
"LDAP") and wants to consume your certificates in the binary DER format. I
guess this is an LDAP thing because everything else I've seen is PEM. But it
is dependent on 
[Convert::ASN1](http://search.cpan.org/~gbarr/Convert-ASN1-0.26/lib/Convert/ASN1.pod "Convert ASN1") so I'm hopeful that it knows
what to do with all those [ASN.1](http://tools.ietf.org/html/rfc4913 "ASN.1")
layers that are hidden in the guts of a certificate. I'll let you know how it
goes. Here's that stonker.

```
Certificate:  
Data:  
Version: 3 (0x2)  
Serial Number:  
47:4f:4f:50:01:70  
Signature Algorithm: sha1WithRSAEncryption  
Issuer: C=US, O=Google Inc, CN=Google Internet Authority  
Validity  
Not Before: Aug 16 11:37:16 2012 GMT  
Not After : Jun 7 19:43:27 2013 GMT  
Subject: C=US, ST=California, O=Google Inc, CN=google.com  
Subject Public Key Info:  
Public Key Algorithm: rsaEncryption  
Public-Key: (1024 bit)  
Modulus:  
00:b5:4e:3d:07:0f:f0:57:3a:aa:68:57:9d:1a:9b:  
1b:dc:55:2f:aa:28:02:00:35:3a:3a:3b:17:00:2e:  
ac:17:2d:49:f5:b2:f7:4f:d7:93:6c:84:ed:9a:d1:  
a0:e0:81:64:7b:4f:67:78:bf:52:ba:d3:4c:d1:c2:  
7e:67:16:fd:7f:62:f7:88:86:1b:ea:1c:38:2a:e8:  
58:d2:04:11:45:67:50:73:30:49:64:6a:79:de:e3:  
af:4d:8b:37:1f:ca:ca:13:dd:9e:76:7e:03:54:bf:  
50:c0:bb:6f:d9:4d:34:8b:66:7e:fd:b3:43:21:c7:  
4c:dc:86:ae:c4:53:b0:fa:db  
Exponent: 65537 (0x10001)  
X509v3 extensions:  
X509v3 Basic Constraints: critical  
CA:FALSE  
X509v3 Extended Key Usage:  
TLS Web Server Authentication, TLS Web Client Authentication  
X509v3 Subject Key Identifier:  
FD:DE:A8:2D:76:DB:A4:74:C1:62:D9:D3:4B:AD:AB:8B:DD:89:7D:78  
X509v3 Authority Key Identifier:  
keyid:BF:C0:30:EB:F5:43:11:3E:67:BA:9E:91:FB:FC:6A:DA:E3:6B:12:24

X509v3 CRL Distribution Points:

Full Name:  
URI:http://www.gstatic.com/GoogleInternetAuthority/GoogleInternetAuthority.crl

Authority Information Access:  
CA Issuers -
URI:http://www.gstatic.com/GoogleInternetAuthority/GoogleInternetAuthority.crt

X509v3 Subject Alternative Name:  
DNS:google.com, DNS:*.google.com, DNS:*.youtube.com, DNS:youtube.com,
DNS:*.youtube-nocookie.com, DNS:youtu.be, DNS:*.ytimg.com, DNS:*.android.com,
DNS:android.com, DNS:*.googlecommerce.com, DNS:googlecommerce.com,
DNS:*.url.google.com, DNS:*.urchin.com, DNS:urchin.com, DNS:*.google-
analytics.com, DNS:google-analytics.com, DNS:*.cloud.google.com, DNS:goo.gl,
DNS:g.co, DNS:*.gstatic.com, DNS:*.google.ac, DNS:*.google.ad,
DNS:*.google.ae, DNS:*.google.af, DNS:*.google.ag, DNS:*.google.am,
DNS:*.google.as, DNS:*.google.at, DNS:*.google.az, DNS:*.google.ba,
DNS:*.google.be, DNS:*.google.bf, DNS:*.google.bg, DNS:*.google.bi,
DNS:*.google.bj, DNS:*.google.bs, DNS:*.google.by, DNS:*.google.ca,
DNS:*.google.cat, DNS:*.google.cc, DNS:*.google.cd, DNS:*.google.cf,
DNS:*.google.cg, DNS:*.google.ch, DNS:*.google.ci, DNS:*.google.cl,
DNS:*.google.cm, DNS:*.google.cn, DNS:*.google.co.ao, DNS:*.google.co.bw,
DNS:*.google.co.ck, DNS:*.google.co.cr, DNS:*.google.co.hu,
DNS:*.google.co.id, DNS:*.google.co.il, DNS:*.google.co.im,
DNS:*.google.co.in, DNS:*.google.co.je, DNS:*.google.co.jp,
DNS:*.google.co.ke, DNS:*.google.co.kr, DNS:*.google.co.ls,
DNS:*.google.co.ma, DNS:*.google.co.mz, DNS:*.google.co.nz,
DNS:*.google.co.th, DNS:*.google.co.tz, DNS:*.google.co.ug,
DNS:*.google.co.uk, DNS:*.google.co.uz, DNS:*.google.co.ve,
DNS:*.google.co.vi, DNS:*.google.co.za, DNS:*.google.co.zm,
DNS:*.google.co.zw, DNS:*.google.com.af, DNS:*.google.com.ag,
DNS:*.google.com.ai, DNS:*.google.com.ar, DNS:*.google.com.au,
DNS:*.google.com.bd, DNS:*.google.com.bh, DNS:*.google.com.bn,
DNS:*.google.com.bo, DNS:*.google.com.br, DNS:*.google.com.by,
DNS:*.google.com.bz, DNS:*.google.com.cn, DNS:*.google.com.co,
DNS:*.google.com.cu, DNS:*.google.com.cy, DNS:*.google.com.do,
DNS:*.google.com.ec, DNS:*.google.com.eg, DNS:*.google.com.et,
DNS:*.google.com.fj, DNS:*.google.com.ge, DNS:*.google.com.gh,
DNS:*.google.com.gi, DNS:*.google.com.gr, DNS:*.google.com.gt,
DNS:*.google.com.hk, DNS:*.google.com.iq, DNS:*.google.com.jm,
DNS:*.google.com.jo, DNS:*.google.com.kh, DNS:*.google.com.kw,
DNS:*.google.com.lb, DNS:*.google.com.ly, DNS:*.google.com.mt,
DNS:*.google.com.mx, DNS:*.google.com.my, DNS:*.google.com.na,
DNS:*.google.com.nf, DNS:*.google.com.ng, DNS:*.google.com.ni,
DNS:*.google.com.np, DNS:*.google.com.nr, DNS:*.google.com.om,
DNS:*.google.com.pa, DNS:*.google.com.pe, DNS:*.google.com.ph,
DNS:*.google.com.pk, DNS:*.google.com.pl, DNS:*.google.com.pr,
DNS:*.google.com.py, DNS:*.google.com.qa, DNS:*.google.com.ru,
DNS:*.google.com.sa, DNS:*.google.com.sb, DNS:*.google.com.sg,
DNS:*.google.com.sl, DNS:*.google.com.sv, DNS:*.google.com.tj,
DNS:*.google.com.tn, DNS:*.google.com.tr, DNS:*.google.com.tw,
DNS:*.google.com.ua, DNS:*.google.com.uy, DNS:*.google.com.vc,
DNS:*.google.com.ve, DNS:*.google.com.vn, DNS:*.google.cv, DNS:*.google.cz,
DNS:*.google.de, DNS:*.google.dj, DNS:*.google.dk, DNS:*.google.dm,
DNS:*.google.dz, DNS:*.google.ee, DNS:*.google.es, DNS:*.google.fi,
DNS:*.google.fm, DNS:*.google.fr, DNS:*.google.ga, DNS:*.google.ge,
DNS:*.google.gg, DNS:*.google.gl, DNS:*.google.gm, DNS:*.google.gp,
DNS:*.google.gr, DNS:*.google.gy, DNS:*.google.hk, DNS:*.google.hn,
DNS:*.google.hr, DNS:*.google.ht, DNS:*.google.hu, DNS:*.google.ie,
DNS:*.google.im, DNS:*.google.info, DNS:*.google.iq, DNS:*.google.is,
DNS:*.google.it, DNS:*.google.it.ao, DNS:*.google.je, DNS:*.google.jo,
DNS:*.google.jobs, DNS:*.google.jp, DNS:*.google.kg, DNS:*.google.ki,
DNS:*.google.kz, DNS:*.google.la, DNS:*.google.li, DNS:*.google.lk,
DNS:*.google.lt, DNS:*.google.lu, DNS:*.google.lv, DNS:*.google.md,
DNS:*.google.me, DNS:*.google.mg, DNS:*.google.mk, DNS:*.google.ml,
DNS:*.google.mn, DNS:*.google.ms, DNS:*.google.mu, DNS:*.google.mv,
DNS:*.google.mw, DNS:*.google.ne, DNS:*.google.ne.jp, DNS:*.google.net,
DNS:*.google.nl, DNS:*.google.no, DNS:*.google.nr, DNS:*.google.nu,
DNS:*.google.off.ai, DNS:*.google.pk, DNS:*.google.pl, DNS:*.google.pn,
DNS:*.google.ps, DNS:*.google.pt, DNS:*.google.ro, DNS:*.google.rs,
DNS:*.google.ru, DNS:*.google.rw, DNS:*.google.sc, DNS:*.google.se,
DNS:*.google.sh, DNS:*.google.si, DNS:*.google.sk, DNS:*.google.sm,
DNS:*.google.sn, DNS:*.google.so, DNS:*.google.st, DNS:*.google.td,
DNS:*.google.tg, DNS:*.google.tk, DNS:*.google.tl, DNS:*.google.tm,
DNS:*.google.tn, DNS:*.google.to, DNS:*.google.tp, DNS:*.google.tt,
DNS:*.google.us, DNS:*.google.uz, DNS:*.google.vg, DNS:*.google.vu,
DNS:*.google.ws, DNS:google.ac, DNS:google.ad, DNS:google.ae, DNS:google.af,
DNS:google.ag, DNS:google.am, DNS:google.as, DNS:google.at, DNS:google.az,
DNS:google.ba, DNS:google.be, DNS:google.bf, DNS:google.bg, DNS:google.bi,
DNS:google.bj, DNS:google.bs, DNS:google.by, DNS:google.ca, DNS:google.cat,
DNS:google.cc, DNS:google.cd, DNS:google.cf, DNS:google.cg, DNS:google.ch,
DNS:google.ci, DNS:google.cl, DNS:google.cm, DNS:google.cn, DNS:google.co.ao,
DNS:google.co.bw, DNS:google.co.ck, DNS:google.co.cr, DNS:google.co.hu,
DNS:google.co.id, DNS:google.co.il, DNS:google.co.im, DNS:google.co.in,
DNS:google.co.je, DNS:google.co.jp, DNS:google.co.ke, DNS:google.co.kr,
DNS:google.co.ls, DNS:google.co.ma, DNS:google.co.mz, DNS:google.co.nz,
DNS:google.co.th, DNS:google.co.tz, DNS:google.co.ug, DNS:google.co.uk,
DNS:google.co.uz, DNS:google.co.ve, DNS:google.co.vi, DNS:google.co.za,
DNS:google.co.zm, DNS:google.co.zw, DNS:google.com.af, DNS:google.com.ag,
DNS:google.com.ai, DNS:google.com.ar, DNS:google.com.au, DNS:google.com.bd,
DNS:google.com.bh, DNS:google.com.bn, DNS:google.com.bo, DNS:google.com.br,
DNS:google.com.by, DNS:google.com.bz, DNS:google.com.cn, DNS:google.com.co,
DNS:google.com.cu, DNS:google.com.cy, DNS:google.com.do, DNS:google.com.ec,
DNS:google.com.eg, DNS:google.com.et, DNS:google.com.fj, DNS:google.com.ge,
DNS:google.com.gh, DNS:google.com.gi, DNS:google.com.gr, DNS:google.com.gt,
DNS:google.com.hk, DNS:google.com.iq, DNS:google.com.jm, DNS:google.com.jo,
DNS:google.com.kh, DNS:google.com.kw, DNS:google.com.lb, DNS:google.com.ly,
DNS:google.com.mt, DNS:google.com.mx, DNS:google.com.my, DNS:google.com.na,
DNS:google.com.nf, DNS:google.com.ng, DNS:google.com.ni, DNS:google.com.np,
DNS:google.com.nr, DNS:google.com.om, DNS:google.com.pa, DNS:google.com.pe,
DNS:google.com.ph, DNS:google.com.pk, DNS:google.com.pl, DNS:google.com.pr,
DNS:google.com.py, DNS:google.com.qa, DNS:google.com.ru, DNS:google.com.sa,
DNS:google.com.sb, DNS:google.com.sg, DNS:google.com.sl, DNS:google.com.sv,
DNS:google.com.tj, DNS:google.com.tn, DNS:google.com.tr, DNS:google.com.tw,
DNS:google.com.ua, DNS:google.com.uy, DNS:google.com.vc, DNS:google.com.ve,
DNS:google.com.vn, DNS:google.cv, DNS:google.cz, DNS:google.de, DNS:google.dj,
DNS:google.dk, DNS:google.dm, DNS:google.dz, DNS:google.ee, DNS:google.es,
DNS:google.fi, DNS:google.fm, DNS:google.fr, DNS:google.ga, DNS:google.ge,
DNS:google.gg, DNS:google.gl, DNS:google.gm, DNS:google.gp, DNS:google.gr,
DNS:google.gy, DNS:google.hk, DNS:google.hn, DNS:google.hr, DNS:google.ht,
DNS:google.hu, DNS:google.ie, DNS:google.im, DNS:google.info, DNS:google.iq,
DNS:google.is, DNS:google.it, DNS:google.it.ao, DNS:google.je, DNS:google.jo,
DNS:google.jobs, DNS:google.jp, DNS:google.kg, DNS:google.ki, DNS:google.kz,
DNS:google.la, DNS:google.li, DNS:google.lk, DNS:google.lt, DNS:google.lu,
DNS:google.lv, DNS:google.md, DNS:google.me, DNS:google.mg, DNS:google.mk,
DNS:google.ml, DNS:google.mn, DNS:google.ms, DNS:google.mu, DNS:google.mv,
DNS:google.mw, DNS:google.ne, DNS:google.ne.jp, DNS:google.net, DNS:google.nl,
DNS:google.no, DNS:google.nr, DNS:google.nu, DNS:google.off.ai, DNS:google.pk,
DNS:google.pl, DNS:google.pn, DNS:google.ps, DNS:google.pt, DNS:google.ro,
DNS:google.rs, DNS:google.ru, DNS:google.rw, DNS:google.sc, DNS:google.se,
DNS:google.sh, DNS:google.si, DNS:google.sk, DNS:google.sm, DNS:google.sn,
DNS:google.so, DNS:google.st, DNS:google.td, DNS:google.tg, DNS:google.tk,
DNS:google.tl, DNS:google.tm, DNS:google.tn, DNS:google.to, DNS:google.tp,
DNS:google.tt, DNS:google.us, DNS:google.uz, DNS:google.vg, DNS:google.vu,
DNS:google.ws, DNS:*.googleapis.cn  
Signature Algorithm: sha1WithRSAEncryption  
c0:a8:27:9e:20:b8:c5:de:9a:32:0a:4f:e3:8b:9b:10:8b:06:  
73:31:ac:91:75:68:dd:d5:1a:eb:23:86:77:2e:78:49:99:9b:  
84:4e:40:0b:50:08:c5:81:21:f2:a6:55:a1:40:27:2f:5f:93:  
c5:0d:0a:51:ff:49:29:4e:2d:80:c6:5e:a5:bb:ca:df:cb:39:  
29:0c:ca:28:18:a7:1c:c3:43:ff:2e:22:a8:df:41:91:7c:c4:  
ba:7c:63:ce:d8:71:46:73:d7:6b:d3:12:a1:93:c0:8d:44:ce:  
25:da:c1:53:05:76:d7:c8:05:c3:2f:62:95:07:36:a2:04:ee:  
b4:15  
```

:calendar: Posted on October 8, 2012
