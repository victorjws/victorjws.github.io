---
title: "sqlalchemy.InvalidRequestError: object load error"
date: 2021-09-24 01:00:00 +0900
categories: Problem-Solving
tags: Python SQLAlchemy flask-sqlalchemy
---
## 들어가며
업무중에 server report에 본적 없던 아래와 같은 오류가 발생해 원인을 분석하게 되었다.
```
sqlalchemy.exc.InvalidRequestError: Row with identity key (<class '__main__.User'>, (1,), None) can't be loaded into an object; the polymorphic discriminator column 'User.user_type' refers to mapped class SubUserA->SubUserA, which is not a sub-mapper of the requested mapped class SubUserB->SubUserB
```

## 원인
확인해본 결과 database에 data가 잘못 들어가 있었던 것이 근본적인 원인이 되었다.
간단히 설명하자면 parent class에서 여러 sub class를 구분할 수 있는 column에 'a'라는 data가 들어가 있어야 하는데 'b'라고 잘못 들어가 있었다.

## 재현
flask, flask_sqlalchemy library를 이용해 간단히 구현한 테스트용 app이다.
```python
from flask import Flask, jsonify
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()


class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_type = db.Column(db.String(20), nullable=False)
    __tablename__ = "User"
    __mapper_args__ = {
        "polymorphic_identity": __tablename__,
        "polymorphic_on": user_type,
    }


class SubUserA(User):
    id = db.Column(db.Integer, db.ForeignKey("User.id"), primary_key=True)
    __tablename__ = "SubUserA"
    __mapper_args__ = {"polymorphic_identity": __tablename__}


class SubUserB(User):
    id = db.Column(db.Integer, db.ForeignKey("User.id"), primary_key=True)
    __tablename__ = "SubUserB"
    __mapper_args__ = {"polymorphic_identity": __tablename__}


app = Flask(__name__)
app.config["SQLALCHEMY_DATABASE_URI"] = "mysql+pymysql://test_id:test_pw@localhost:3306/test_database"
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = True


@app.route("/test")
def test():
    SubUserB.query.all()
    return jsonify({"test": "success"})


if __name__ == "__main__":
    db.init_app(app)
    app.run(port=5000)
```
User를 상속받은 SubUserA, SubUserB가 있으며 user_type으로 SubUserA인지 SubUserB인지 구분하도록 구현되어있다.
Database 또한 아래 SQL과 같이 User, SubUserA, SubUserB, Request 각 table로 이루어져 있으며, 
User의 user_type은 복수의 값을 가질 수 없으므로 SubUserA나 SubUserB 둘 중 하나만 될 수 있다.
```sql
CREATE TABLE `User` (
    `id` INT PRIMARY KEY,
    `user_type` VARCHAR(20) NOT NULL
);

CREATE TABLE `SubUserA` (
    `id` INT NOT NULL,
    FOREIGN KEY (`id`) REFERENCES `user` (`id`)
);

CREATE TABLE `SubUserB` (
    `id` INT NOT NULL,
    FOREIGN KEY (`id`) REFERENCES `user` (`id`)
);
```

이 상태에서 Database의 User table에 user_type은 'SubUserA'로 저장하고 SubUserB에 User table의 id와 같은 id 값을 저장한다.
```sql
INSERT INTO `User` VALUES (1, 'SubUserA');
INSERT INTO `SubUserB` VALUES (1);
```

아래의 명령어를 실행하면 처음에 본 에러가 발생하게 된다.
```commandline
curl "http://localhost:5000/v2/ping"
```

## 분석
SQLAlchemy library의 sqlalchemy/orm/loading.py의
_decorate_polymorphic_switch() 내부 함수인 polymorphic_instance()를 확인해보면
query로 가져온 row들을 object로 변환하는 과정에서 error가 발생했음을 볼 수 있다.
```python
def polymorphic_instance(row):
    discriminator = getter(row)  # row의 column 중 polymorphic_on으로 정의한 값을 가져옴
    if discriminator is not None:
        _instance = polymorphic_instances[discriminator]  # discriminator와 query 생성시 사용한 mapper를 비교하여 동일하거나 부모 클래스인지 확인
        if _instance:
            return _instance(row)
        elif _instance is False:
            identitykey = ensure_no_pk(row)  # primary_key가 null인지 확인, null일 경우 None 반환

            if identitykey:
                raise sa_exc.InvalidRequestError(
                    "Row with identity key %s can't be loaded into an "
                    "object; the polymorphic discriminator column '%s' "
                    "refers to %s, which is not a sub-mapper of "
                    "the requested %s"
                    % (
                        identitykey,
                        polymorphic_on,
                        mapper.polymorphic_map[discriminator],
                        mapper,
                    )
                )
            else:
                return None
        else:
            return instance_fn(row)
    else:
        identitykey = ensure_no_pk(row)

        if identitykey:
            raise sa_exc.InvalidRequestError(
                "Row with identity key %s can't be loaded into an "
                "object; the polymorphic discriminator column '%s' is "
                "NULL" % (identitykey, polymorphic_on)
            )
        else:
            return None
```

## References
- [SQLAlchemy repository](https://github.com/sqlalchemy/sqlalchemy)
