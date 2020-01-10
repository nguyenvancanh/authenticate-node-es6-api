Sử dụng Json web token để authenticate cho Node API

Trong bài viết này chúng ta sẽ làm một số công việc chính sau đây:

- Cài đặt môi trường dev với việc sử dụng express server
- Tạo router và controller cơ bản
- Sử dụng router và controller để thêm mới người dùng và làm chức năng đăng nhập
- Tạo một route và controller để lấy tất cả user

Nâng cao hơn 1 chút:

- Thêm middleware cho route lấy toàn bộ danh sách người dùng: Chỉ người dùng nào có role admin và token thỏa mãn mới có thể truy cập

## Setup

Trước khi bắt đầu, có một vài điểm cần chú ý như sau:

Cấu trúc thư mục project của chúng ta có dạng như sau:

```
├── config.js
├── controllers
│   └── users.js
├── index.js
├── models
│   └── users.js
├── routes
│   ├── index.js
│   └── users.js
├── utils.js
```

Bạn có thể tạo nhanh thư mục project với các lệnh:

```
$ mkdir -p jwt-node-auth/{controllers/users.js,models/users.js,routes/index.js,routes/users.js}
$ cd jwt-node-auth
$ touch utils.js && touch config.js && touch index.js
```

Chú ý là bạn phải cài nodejs nhé. Sau khi cài đặt nodejs, cài thêm các package dependencies cần thiết để chạy project. 

Chạy câu lệnh sau để khởi tạo file package.jon

```
npm init --yes
```
Chạy câu lệnh sau để cài đạt toàn bộ dependencies cần thiết

```
$ npm install express body-parser bcrypt dotenv jsonwebtoken mongoose  --save
$ npm install morgan nodemon cross-env  --save-dev
```

Giải thích đôi chút về những thư viện chúng ta vừa cài

- body-parser: Thêm toàn bộ thông thi chúng ta truyền tới API vào trong object request.body
- bcrypt: Sử dụng để mã hóa password trước khi lưu vào DB
- dotenv: Sử dụng để load toàn bộ biến môi trường trong file .env, tất nhiên là sẽ được bảo mật
- jsonwebtoken: Sử dụng để sign and verify JSON web token
- mongoose: Sử dụng để làm việc với mongo DB
- morgan: Log tất cả các request  chúng ta tạo ra trong môi trường dev
- nodemon: Sử dụng để restart server một cách tự động khi chúng ta sử code

## **Biến môi trường**

Khai báo các biến vào trong file .env như sau

```
JWT_SECRET=addjsonwebtokensecretherelikeQuiscustodietipsoscustodes
MONGO_LOCAL_CONN_URL=mongodb://127.0.0.1:27017/node-jwt
MONGO_DB_NAME=auth-with-jwts
```

## **Khởi tạo server**

Trước tiên, thêm vào file package.json

```
"scripts": {
    "dev": "cross-env NODE_ENV=development nodemon index.js"
  },
```

Sau đó chạy lệnh _npm run dev_
Với việc này, development sẽ tự động set value cho biến NODE_ENV trong process object. command _nodemon index.js_ sẽ cho phép nodemon restart server mỗi khi chúng ta thay đổi trong thư mục code

Tiếp theo, là file config.js

```
module.exports = {
  development: {
    port: process.env.PORT || 3000
  }
}
```

Sau đó là file index.js

```
require('dotenv').config(); // Sets up dotenv as soon as our application starts

const express = require('express'); 
const logger = require('morgan');
const bodyParser = require('body-parser');

const app = express();
const router = express.Router();

const environment = process.env.NODE_ENV; // development
const stage = require('./config')[environment];

app.use(bodyParser.json());
app.use(bodyParser.urlencoded({
  extended: true
}));

if (environment !== 'production') {
  app.use(logger('dev'));
}

app.use('/api/v1', (req, res, next) => {
  res.send('Hello');
  next();
});

app.listen(`${stage.port}`, () => {
  console.log(`Server now listening at localhost:${stage.port}`);
});

module.exports = app;
```

Chạy lệnh npm run dev ở thư mục root, và đảm bảo rằng chữ _Hello_ sẽ được in ra trên màn hình khi bạn truy cập vào đường link (localhost:3000/api/v1)[localhost:3000/api/v1] 

## Add user functionality

Sửa file **index.js** với đoạn code

```
const routes = require('./routes/index.js');

app.use('/api/v1', routes(router));
```

**controllers/users.js**

```
module.exports = {
  add: (req, res) => {
    return;
  }
}
```

## Route setup

**routes/index.js**

```
const users = require('./users');

module.exports = (router) => {
  users(router);
  return router;
};
```

**routes/users.js**

```
const controller = require('../controllers/users');

module.exports = (router) => {
  router.route('/users')
    .post(controller.add);
};
```

Đoạn code trên định nghĩa route cho method add trong user controller vào trong reoute /users với method POST

## **User Model**

**models/users.js**

```
const mongoose = require('mongoose');
const bcrypt = require('bcrypt');

const environment = process.env.NODE_ENV;
const stage = require('./config')[environment];

// schema maps to a collection
const Schema = mongoose.Schema;

const userSchema = new Schema({
  name: {
    type: 'String',
    required: true,
    trim: true,
    unique: true
  },
  password: {
    type: 'String',
    required: true,
    trim: true
  }
});

module.exports = mongoose.model('User', userSchema);
```
Để thêm được user vào collection , thì cần phải truyền vào name và password với dạng string

## Hashing users passwords

Sử dụng bscrypt để mã hóa mật khẩu của user trước khi lưu vào DB. Thêm một line vào file config để định nghĩa số lần bạn muốn Salt password

```
 module.exports = {
  development: {
    port: process.env.PORT || 3000,
    saltingRounds: 10
  }
}
```

Sửa lại file models/users.js

```
// encrypt password before save
userSchema.pre('save', function(next) {
  const user = this;
  if(!user.isModified || !user.isNew) { // don't rehash if it's an old user
    next();
  } else {
    bcrypt.hash(user.password, stage.saltingRounds, function(err, hash) {
      if (err) {
        console.log('Error hashing password for user', user.name);
        next(err);
      } else {
        user.password = hash;
        next();
      }
    });
  }
});
```

Như vậy là password của bạn đã được mã hóa, công việc tiếp theo sẽ là sửa controller để thêm user

**controllers/users.js**

```
const mongoose = require('mongoose');
const User = require('../models/users');

const connUri = process.env.MONGO_LOCAL_CONN_URL;

module.exports = {
  add: (req, res) => {
    mongoose.connect(connUri, { useNewUrlParser : true }, (err) => {
      let result = {};
      let status = 201;
      if (!err) {
        const { name, password } = req.body;
        const user = new User({ name, password }); // document = instance of a model
        // TODO: We can hash the password here before we insert instead of in the model
        user.save((err, user) => {
          if (!err) {
            result.status = status;
            result.result = user;
          } else {
            status = 500;
            result.status = status;
            result.error = err;
          }
          res.status(status).send(result);
        });
      } else {
        status = 500;
        result.status = status;
        result.error = err;
        res.status(status).send(result);
      }
    });
  },
}
```

Trong đoạn code trên, chúng ta connect tới mongodb sau đó lấy name và password được cung cập trong the request bằng cách sử dụng object request.body. Sau đó tạo user mới bằng việc gọi _new_ trong model. 

```
const name = req.body.name;
const password = req.body.password;

let user = new User();
user.name = name;
user.password = password;

user.save((err, user) => { ... }
```

## Login function

**routes/users.js**

```
const controller = require('../controllers/users');

module.exports = (router) => {
  router.route('/users')
    .post(controller.add);

  router.route('/login')
    .post(controller.login)
};
```

importe thêm bscrypt vào user controoler

```
const bcrypt = require('bcrypt');
```
Sau đó, thêm login controller để handel method login như sau:

```
login: (req, res) => {
    const { name, password } = req.body;

    mongoose.connect(connUri, { useNewUrlParser: true }, (err) => {
      let result = {};
      let status = 200;
      if(!err) {
        User.findOne({name}, (err, user) => {
          if (!err && user) {
            // We could compare passwords in our model instead of below
            bcrypt.compare(password, user.password).then(match => {
              if (match) {
                result.status = status;
                result.result = user;
              } else {
                status = 401;
                result.status = status;
                result.error = 'Authentication error';
              }
              res.status(status).send(result);
            }).catch(err => {
              status = 500;
              result.status = status;
              result.error = err;
              res.status(status).send(result);
            });
          } else {
            status = 404;
            result.status = status;
            result.error = err;
            res.status(status).send(result);
          }
        });
      } else {
        status = 500;
        result.status = status;
        result.error = err;
        res.status(status).send(result);
      }
    });
  }
```

Thêm token để authenticatinon 

```
 // controllers/users.js
 const jwt = require('jsonwebtoken');
```
```
login: (req, res) => {
    const { name, password } = req.body;

    mongoose.connect(connUri, { useNewUrlParser: true }, (err) => {
      let result = {};
      let status = 200;
      if(!err) {
        User.findOne({name}, (err, user) => {
          if (!err && user) {
            // We could compare passwords in our model instead of below as well
            bcrypt.compare(password, user.password).then(match => {
              if (match) {
                status = 200;
                // Create a token
                const payload = { user: user.name };
                const options = { expiresIn: '2d', issuer: 'https://scotch.io' };
                const secret = process.env.JWT_SECRET;
                const token = jwt.sign(payload, secret, options);

                // console.log('TOKEN', token);
                result.token = token;
                result.status = status;
                result.result = user;
              } else {
                status = 401;
                result.status = status;
                result.error = `Authentication error`;
              }
              res.status(status).send(result);
            }).catch(err => {
              status = 500;
              result.status = status;
              result.error = err;
              res.status(status).send(result);
            });
          } else {
            status = 404;
            result.status = status;
            result.error = err;
            res.status(status).send(result);
          }
        });
      } else {
        status = 500;
        result.status = status;
        result.error = err;
        res.status(status).send(result);
      }
    });
  }
```
