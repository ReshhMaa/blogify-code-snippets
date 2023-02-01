## Codes
###### This are some code snippets that can be copied to your code
### blogify/user_posts/forms.py
##### Some necessary imports 
```
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField, TextAreaField
from wtforms.validators import DataRequired, Email, EqualTo
from wtforms import ValidationError
```
```
from flask_login import current_user
from blogify.models import User
```
##### Class for login form
```
class LoginForm(FlaskForm):
    email = StringField('Email',validators=[DataRequired(),Email()])
    password = PasswordField('Password',validators=[DataRequired()])
    submit = SubmitField('Log In')

```
##### Class for Registration form 
```
class RegistrationForm(FlaskForm):
    email = StringField('Email',validators=[DataRequired(),Email()])
    username = StringField('UserName',validators=[DataRequired()])
    password = PasswordField('Password',validators=[DataRequired(),EqualTo('pass_confirm',message='Passwords must match!')])
    pass_confirm = PasswordField('Confirm Password',validators=[DataRequired()])
    submit = SubmitField('Register!')
```
##### Email validation inside class ResgistrationForm
```
    def check_email(self,field):
        if User.query.filter_by(email=field.data).first():
            raise ValidationError('Your email has been registered already!')
```
##### Username validation inside class RegistrationForm
```
    def check_username(self,field):
        if User.query.filter_by(username=field.data).first():
            raise ValidationError('Your username has been registered already!')
```
##### Class for Blog posts form
```
class BlogPostForm(FlaskForm):
    title = StringField('Title', validators=[DataRequired()])
    text = TextAreaField('Text', validators=[DataRequired()])
    submit = SubmitField('Post')
```
### blogify/__init__.py
````
````
##### Some necessary imports
```
from flask import Flask, render_template
import os
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager

```
##### app initializing and setting secret key
```
app = Flask(__name__)
SECRET_KEY = os.urandom(32)
app.config['SECRET_KEY'] = SECRET_KEY
```
##### Database setup
```
basedir = os.path.abspath(os.path.dirname(__file__))
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///'+os.path.join(basedir,'data.sqlite')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
```
```
db = SQLAlchemy(app)
Migrate(app,db)
```
##### Login Configs
```
login_manager = LoginManager()

login_manager.init_app(app)
login_manager.login_view = 'users_posts.login'
```
##### Error Handling 
```
@app.errorhandler(404)
def error_404(error):
  return render_template('error_pages/404.html'), 404

@app.errorhandler(403)
def error_403(error):
  return render_template('error_pages/403.html'), 403
```
##### creating blueprints
```
from blogify.core.views import core
from blogify.users_posts.views import users_posts

app.register_blueprint(core)
app.register_blueprint(users_posts)
```
### blogify/models.py
 
