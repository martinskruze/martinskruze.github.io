---
layout: post
title:  recipe - create OBS source with LIVE data from server
date:   2018-02-06 14:31:00 +0200
categories: recipes
---
## needed software:
1. [Open Broadcaster Software](https://obsproject.com/)

2. add scenes -> browser source
3. web page data can be changed with ajax or can be autorefreshed with javascript (brutal):
```javascript
function reload(){
    location.href=location.href
}
setInterval('reload()',14000);
```

### to carry over scenes in obs:
scene collection export - big question - can we generate this file?

### test example:
```html
<!DOCTYPE html>
	<head>
		<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
		<title>browser source test</title>
		<script type="text/javascript">
			function setData(){
				document.getElementById("data").innerHTML = Date.now();
			}
			setInterval('setData()',1000);
		</script>
	</head>
	<body>
		<h1 id="data"></h1>
		<script type="text/javascript">
			setData();
		</script>
	</body>
</head>
```
