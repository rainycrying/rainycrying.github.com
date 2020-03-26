运行环境：Ubuntu 18.04 、 Node.js 10.*

安装 Node.js 10.*
添加软件源：

curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -


安装 Node.js 10.* ：

apt-get install -y nodejs

查看 Nodejs 版本：

nodejs -v

获取 Cloudflare WARP 的 AFF ID
进入 Cloudflare WARP ，点击右上角的 设置 按钮，进入 更多设置 - 诊断 ， 客户端配置 里面的 ID 即为 AFF ID 。复制 AFF ID ，下面将用到。



创建 js 脚本
使用如下命令创建 js 脚本，注意将 AFF ID 替换成自己的。循环次数默认为 10 ，即执行一次脚本循环 10 次，增加 10G 流量。可根据自己的需要酌情修改。建议按默认设置即可：

vim cloudflare-warp-plus-aff.js

复制以下内容粘贴并保存：


// Fake register for referrer to get warp plus bandwidth
const referrer = "AFF ID复制到这里";
const timesToLoop = 10; // 循环次数
const retryTimes = 5; // 重试次数

const https = require("https");
const zlib = require("zlib");

async function init() {
  for (let i = 0; i < timesToLoop; i++) {
    if (await run()) {
      console.log(i + 1, "OK");
    } else {
      console.log(i + 1, "Error");
      for (let r = 0; r < retryTimes; r++) {
        if (await run()) {
          console.log(i + 1, "Retry #" + (r + 1), "OK");
          break;
        } else {
          console.log(i + 1, "Retry #" + (r + 1), "Error");
          if (r === retryTimes - 1) {
            return;
          }
        }
      }
    }
  }
}

async function run() {
  return new Promise(resolve => {
    const install_id = genString(11);
    const postData = JSON.stringify({
      key: `${genString(43)}=`,
      install_id: install_id,
      fcm_token: `${install_id}:APA91b${genString(134)}`,
      referrer: referrer,
      warp_enabled: false,
      tos: new Date().toISOString().replace("Z", "+07:00"),
      type: "Android",
      locale: "zh_CN"
    });

    const options = {
      hostname: "api.cloudflareclient.com",
      port: 443,
      path: "/v0a745/reg",
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Host: "api.cloudflareclient.com",
        Connection: "Keep-Alive",
        "Accept-Encoding": "gzip",
        "User-Agent": "okhttp/3.12.1",
        "Content-Length": postData.length
      }
    };

    const req = https.request(options, res => {
      const gzip = zlib.createGunzip();
      // const buffer = [];
      res.pipe(gzip);
      gzip
        .on("data", function(data) {
          // buffer.push(data.toString());
        })
        .on("end", function() {
          // console.dir(JSON.parse(buffer.join("")));
          resolve(true);
        })
        .on("error", function(e) {
          // console.error(e);
          resolve(false);
        });
    });

    req.on("error", error => {
      // console.error(error);
      resolve(false);
    });

    req.write(postData);
    req.end();
  });
}

function genString(length) {
  // https://stackoverflow.com/a/1349426/11860316
  let result = "";
  const characters = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
  const charactersLength = characters.length;
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * charactersLength));
  }
  return result;
}

init();





执行 js 脚本
使用如下命令执行脚本：

node cloudflare-warp-plus-aff.js


脚本运行后返回 OK 即表示成功。刷新一下 Cloudflare WARP ，看看 WARP+ 的流量是不是增加了。
