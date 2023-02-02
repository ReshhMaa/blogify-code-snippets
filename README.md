## Code Snippets

This are some code snippets that can be copied to your code

### 1. Forms
File Route: blogify/user_posts/forms.py

##### Some necessary imports 

```python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField, TextAreaField
from wtforms.validators import DataRequired, Email, EqualTo
from wtforms import ValidationError
```

```python
from flask_login import current_user
from blogify.models import User
```

##### Class for Login form

```python
class LoginForm(FlaskForm):
    email = StringField('Email',validators=[DataRequired(),Email()])
    password = PasswordField('Password',validators=[DataRequired()])
    submit = SubmitField('Log In')
```

##### Class for Registration form 

```python
class RegistrationForm(FlaskForm):
    email = StringField('Email',validators=[DataRequired(),Email()])
    username = StringField('UserName',validators=[DataRequired()])
    password = PasswordField('Password',validators=[DataRequired(),EqualTo('pass_confirm',message='Passwords must match!')])
    pass_confirm = PasswordField('Confirm Password',validators=[DataRequired()])
    submit = SubmitField('Register!')
```

##### Email validation inside class ResgistrationForm

```python
    def check_email(self,field):
        if User.query.filter_by(email=field.data).first():
            raise ValidationError('Your email has been registered already!')
```

##### Username validation inside class RegistrationForm

```python
    def check_username(self,field):
        if User.query.filter_by(username=field.data).first():
            raise ValidationError('Your username has been registered already!')
```

##### Class for Blog Posts Form

```python
class BlogPostForm(FlaskForm):
    title = StringField('Title', validators=[DataRequired()])
    text = TextAreaField('Text', validators=[DataRequired()])
    submit = SubmitField('Post')
```

### 2. 
File Route: blogify/\_\_init\_\_.py

##### Some necessary imports

```python
from flask import Flask, render_template
import os
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager
```

##### App initializing and Setting Secret Key

```python
app = Flask(__name__)
SECRET_KEY = os.urandom(32)
app.config['SECRET_KEY'] = SECRET_KEY
```

##### Database setup

```python
basedir = os.path.abspath(os.path.dirname(__file__))
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///'+os.path.join(basedir,'data.sqlite')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
```

```python
db = SQLAlchemy(app)
Migrate(app,db)
```

##### Login Configs

```python
login_manager = LoginManager()

login_manager.init_app(app)
login_manager.login_view = 'users_posts.login'
```

##### Error Handling 

```python
@app.errorhandler(404)
def error_404(error):
  return render_template('error_pages/404.html'), 404

@app.errorhandler(403)
def error_403(error):
  return render_template('error_pages/403.html'), 403
```

##### Creating Blueprints

```python
from blogify.core.views import core
from blogify.users_posts.views import users_posts

app.register_blueprint(core)
app.register_blueprint(users_posts)
```

### 3. Models
File Route: blogify/models.py

##### Some necessary imports

```python
from blogify import db,login_manager
from werkzeug.security import generate_password_hash,check_password_hash
from flask_login import UserMixin
from datetime import datetime
```

##### Routing for Authentication

```python
@login_manager.user_loader
def load_user(user_id):
    return User.query.get(user_id)
```

##### Class for Storing User Info

```python
class User(db.Model,UserMixin):

    __tablename__ = 'users'

    id = db.Column(db.Integer,primary_key=True)
    email = db.Column(db.String(64),unique=True,index=True)
    username = db.Column(db.String(64),unique=True,index=True)
    password_hash = db.Column(db.String(128))

    posts = db.relationship('BlogPost',backref='author',lazy=True)

    def __init__(self,email,username,password):
        self.email = email
        self.username = username
        self.password_hash = generate_password_hash(password)

    def check_password(self,password):
        return check_password_hash(self.password_hash,password)

    def __repr__(self):
        return f"Username {self.username}"
```

##### Class for Storing Blog Post Info

```python
class BlogPost(db.Model):

    users = db.relationship(User)

    id = db.Column(db.Integer,primary_key=True)
    user_id = db.Column(db.Integer,db.ForeignKey('users.id'),nullable=False)

    date = db.Column(db.DateTime,nullable=False,default=datetime.utcnow)
    title = db.Column(db.String(140),nullable=False)
    text = db.Column(db.Text,nullable=False)

    def __init__(self,title,text,user_id):
        self.title = title
        self.text = text
        self.user_id = user_id

    def __repr__(self):
        return f"Post ID: {self.id} -- Date: {self.date} --- {self.title}"
```
