# node-imap-demo

## 本篇文章只有記錄 IMAP 的使用方式，POP3 代補

#主要套件

[ node-imap ]

https://github.com/mscdex/node-imap

網友寫法參考( 沒有全對 ): https://gist.github.com/jascination/85719030bf4b9abc7203

[ MailParser ]

https://nodemailer.com/extras/mailparser/

# 次要套件

[bufferhelper]

https://github.com/JacksonTian/bufferhelper

# 主程式 ( 尚未重構，一堆垃圾 code )
``` JS
var Imap = require('imap'),
  inspect = require('util').inspect;

var fs = require('fs'), fileStream;
const simpleParser = require('mailparser').simpleParser;
var BufferHelper = require('bufferhelper');
var source = '';

var LineBot = require('./linebot');
var logger = require('./logger');

var header_buffer_helper = new BufferHelper();
var text_buffer_helper = new BufferHelper();
var source_buffer_helper = new BufferHelper();

var bot = new LineBot({
  channelId: '',
  channelSecret: '',
  channelAccessToken: ''
});

var userId = '';

//  Connection object
var imap = new Imap({
  user: '',
  password: '',
  host: '',
  port: 143,
  tls: false
});

var seqno = -1;
var seqno_source = '';

function openInbox(cb) {
  imap.openBox('INBOX', true, cb);
}

imap.once('ready', function () {
  console.log('Imap Ready...');

  openInbox(function (err, box) {
    if (err) throw err;
    console.log('Imap OpenInBox...');
    bot.sendText(userId, 'XXX Mail Inbox Startup');
  });
});

imap.once('error', function (err) {
  console.log('Imap Error...');
  console.log(err);
    bot.sendText(userId, 'Imap Error:\n' + err);
});

imap.once('end', function () {
  console.log('Imap End...');
});

imap.once('close', function () {
  console.log('Imap Close...');
  bot.sendText(userId, 'XXX Mail Inbox Shutdown');
});

/**
 * Emitted when the server issues an alert (e.g. "the server is going down for maintenance").
 * 當服務器發出警報時發出（例如“服務器正在進行維護”）。
 */
imap.on('alert', function (message) {
  console.log('Imap Alert...');
  console.log(message);
  bot.sendText(userId, 'Imap Alert:\n' + message);
});

/**
 * Emitted when new mail arrives in the currently open mailbox.
 * 當新郵件到達當前打開的郵箱時發出。
 * 
 * Server 收到信的通知
 */
imap.on('mail', function (numNewMsgs) {
  console.log('Imap Mail...');
  console.log(numNewMsgs);
  // seqno = 0;
  // seqno = 140;
  // numNewMsgs = 8;
  if (seqno === -1) {
    seqno = numNewMsgs;
    seqno_source = `${seqno}:*`;
  } else {
    seqno_source = `${seqno + 1}:*`;
  }

  fetch();
});

/**
 * Emitted when message metadata (e.g. flags) changes externally.
 * 當消息元數據（例如標誌）外部改變時發出。
 * 
 * 當 Client 從 Server 收信的通知
 */
imap.on('update', function (seqno, info) {
  console.log('Imap Update...');
  console.log(seqno);
  console.log(info);
});

/**
 * Emitted when a message was expunged externally. 
 * seqno is the sequence number (instead of the unique UID) of the message that was expunged.
 * If you are caching sequence numbers, 
 * all sequence numbers higher than this value MUST be decremented 
 * by 1 in order to stay synchronized with the server and to keep correct continuity.
 * 消息在外部被清除時發出。 
 * seqno是被清除的消息的序列號（而不是唯一的UID）。 
 * 如果緩存序列號，則高於此值的所有序列號必須減1，以保持與服務器同步並保持正確的連續性。
 * 
 * 當 Client 刪除信件的通知
 */
imap.on('expunge', function (_seqno) {
  console.log('Imap Expunge...');
  console.log(_seqno);
  seqno = seqno - 1;
});

/**
 * Emitted if the UID validity value for the currently open mailbox changes during the current session.
 * 如果當前打開的郵箱的UID有效值在當前會話期間發生更改，則發送。
 */
imap.on('uidvalidity', function (uidvalidity) {
  console.log('Imap Uidvalidity...');
  console.log(uidvalidity);
  bot.sendText(userId, 'Imap Uidvalidity:\n' + uidvalidity);
});

imap.connect();

function fetch() {
  var f = imap.seq.fetch(seqno_source, {
    bodies: '',
    struct: true
  });

  f.on('message', function (msg, seqno) {
    msg.on('body', function (stream, info) {
      simpleParser(stream).then(function (mail) {
        var subject = mail.headers.get('subject');
        var reg = new RegExp('r\\d+ - ', 'gm');
        var reg2 = new RegExp('Re: sus ', 'gm');
        var reg3 = new RegExp('Weekly Report ', 'gm');
        var reg4 = new RegExp('Sections, ', 'gm');

        if (!reg.test(subject) && !reg2.test(subject) && !reg3.test(subject) && !reg4.test(subject)) {
          console.log(`[${mail.date}] 收到新郵件: ${subject}`);
          bot.sendText(userId, '收到新郵件:\n' + subject);
        }
        // console.log('==== level 1 ====');
        // for (var prop_name in mail) {
        //   console.log(prop_name);
        // }
        // console.log('==== headers ====');
        // for (var [name] of mail.headers) {
        //   console.log(name);
        // }
        // fs.writeFile('./1.ppt', mail.attachments[1].content, null, function (err) {
        //   if (err) throw err;
        //   console.log('It\'s saved!');
        // });
      });
    });

    msg.on('attributes', function (attrs) {
      parseStruct(attrs.struct);
    });

    msg.on('end', function () {
      // console.log('Message End...');
    });
  });

  f.on('error', function (err) {
    console.log('Fetch Error...');
    console.log(err);
    bot.sendText(userId, 'Fetch Error:\n' + err);
  });

  f.once('end', function () {
    console.log('Fetch End...');
    // imap.end();
  });
}


function parseStruct(struct) {
  var part = getPartBodies(struct);

  if ((part.subtype === 'plain') && (part.encoding === 'base64')) {
  } else if ((part.params.charset === 'big5') && (part.encoding === '8bit')) {
    
  } else {

  }
}

function getPartBodies(struct) {
  for (var i = 0; i < struct.length; i++) {
    var _ = struct[i];

    if (Array.isArray(_)) {
      return getPartBodies(_)
    } else if (_.partID && _.subtype === 'plain') {
      return _;
    } else if (_.partID && _.subtype === 'html') {
      return _;
    }
  }
}
```
# Package.json

``` JS
{
  "name": "mail",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "bufferhelper": "^0.2.1",
    "imap": "^0.8.19",
    "mailparser": "^2.0.5"
  }
}
```

# 程式語法技巧

[ inspect ]

const inspect = require('util').inspect;
http://www.runoob.com/nodejs/nodejs-util.html

# 參考資料

一次搞懂SMTP、POP3、IMAP是什麼

https://goo.gl/yj8uvd

# 測試 IMAP、POP3 連線資訊是否可以正常連接

https://www.dotcom-tools.com/email-server-test.aspx

# 如何使用 IMAP、MailParser，以及處理編碼問題

https://cnodejs.org/topic/52f9ea421b80994b0dd18388

# node-imap 標記郵件為已讀

https://cnodejs.org/topic/539eb401c3ee0b5820588af4
