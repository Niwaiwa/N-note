###### tags: `Go`
# Go blog design

Common blog design

## components

* Auth: login, logout
    * jwt - redis
* User: create, get, edit, delete
* Article: create, get, edit, delete

## API

* Auth

| route | method | note |
| -------- | -------- | -------- |
| /login   | POST     |          |
| /logout  | POST     |          |

* User

| route | method | note |
| -------- | -------- | -------- |
| /register | POST    | create user |
| /user     | GET     | get user    |
|           | POST    | update user |
|           | DELETE  | delete user |

* Article

| route | method | note |
| -------- | -------- | -------- |
| /article     | GET      | get articles   |
|              | POST     | create article |
| /article/:id | GET      | get article    |
|              | POST     | update article |
|              | DELETE   | delete article |


## database


* User

| name | type | note |
| -------- | -------- | -------- |
| id       | integer  |          |
| username | string   | unique   |
| account  | string   | unique   |
| password | string   | hash     |
| created_at | date   |          |
| updated_at | date   |          |

* Article

| name | type | note |
| -------- | -------- | -------- |
| id       | integer  |          |
| title    | string   |          |
| content  | string   |          |
| user_id  | integer  |          |
| created_at | date   |          |
| updated_at | date   |          |
