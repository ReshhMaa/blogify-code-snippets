# Flow

/app.py

```python
from blogify import app

if __name__ == '__main__':
    app.run(host='localhost', port=3001, debug=True)
```

/blogify/\_\_init\_\_.py

```python
#imports
from flask import Flask, render_template
import os
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_login import LoginManager

app = Flask(__name__)
SECRET_KEY = os.urandom(32)
app.config['SECRET_KEY'] = SECRET_KEY
```

In same file we added error handling code

```python
# Error Handling
@app.errorhandler(404)
def error_404(error):
  return render_template('error_pages/404.html'), 404

@app.errorhandler(403)
def error_403(error):
  return render_template('error_pages/403.html'), 403
```

Now we create end point for index page and info page

\blogify\core\views.py

For that purpose we create a sample html file in template directory

Name the file `sample.html`

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Document</title>
  </head>
  <body>
    hello hitesh
  </body>
</html>
```

```python
# core/views.py
from flask import render_template,request,Blueprint

core = Blueprint('core',__name__)

@core.route('/')
def index():

    return render_template('sample.html')

@core.route('/info')
def info():
    return render_template('info.html')
```

Now we register the core blueprint in \_\_init\_\_.py at the end

```python
from blogify.core.views import core

app.register_blueprint(core)
```

Now we check if the file is loading on the browser
For that reason use the command
`python app.py`
We check the end point -> index and info

Now we code database structure

- Models
- Database related imports and setup in /blogify/\_\_init\_\_.py
- Three commands

1. Code in Models \blogify\models.py

```python
#models.py
from blogify import db,login_manager
from werkzeug.security import generate_password_hash,check_password_hash
from flask_login import UserMixin
from datetime import datetime

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(user_id)

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

Add DB setup and Login Config in /blogify/\_\_init\_\_.py
Add it above error 404 and 403 handling code

```python
# Database setup
basedir = os.path.abspath(os.path.dirname(__file__))
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///'+os.path.join(basedir,'data.sqlite')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)
Migrate(app,db)

# Login Configs
login_manager = LoginManager()

login_manager.init_app(app)
login_manager.login_view = 'users_posts.login'
```
Add import statement in \core\views.py

```python
from blogify.models import BlogPost
```
To setup the database, run the following commands:

```
flask db init
flask db migrate -m "first migration"
flask db upgrade
```

---- We have completed db setup ----

You can check if the tables are created in data.sqlite
Also a migrations folder must have been created.

---

Forms

1. We code in uesr_posts/views.py

Add imports and create a blueprint

```python
from flask import abort, render_template, url_for, flash, redirect, request, Blueprint
from flask_login import login_user, current_user, logout_user, login_required
from blogify import db
from blogify.models import User, BlogPost
from blogify.users_posts.forms import RegistrationForm, LoginForm, BlogPostForm

users_posts = Blueprint('users_posts', __name__)
```

Register the blueprint in /blogify/\_\_init\_\_.py

> Alert - you know what i mean

```python
from blogify.core.views import core
from blogify.users_posts.views import users_posts

app.register_blueprint(core)
app.register_blueprint(users_posts)
```

Now we edit forms file in \user_posts\forms.py in order to create form structure.

```python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField, TextAreaField
from wtforms.validators import DataRequired, Email, EqualTo
from wtforms import ValidationError

from flask_login import current_user
from blogify.models import User

class LoginForm(FlaskForm):
    email = StringField('Email',validators=[DataRequired(),Email()])
    password = PasswordField('Password',validators=[DataRequired()])
    submit = SubmitField('Log In')

class RegistrationForm(FlaskForm):
    email = StringField('Email',validators=[DataRequired(),Email()])
    username = StringField('UserName',validators=[DataRequired()])
    password = PasswordField('Password',validators=[DataRequired(),EqualTo('pass_confirm',message='Passwords must match!')])
    pass_confirm = PasswordField('Confirm Password',validators=[DataRequired()])
    submit = SubmitField('Register!')

    def check_email(self,field):
        if User.query.filter_by(email=field.data).first():
            raise ValidationError('Your email has been registered already!')

    def check_username(self,field):
        if User.query.filter_by(username=field.data).first():
            raise ValidationError('Your username has been registered already!')

class BlogPostForm(FlaskForm):
    title = StringField('Title', validators=[DataRequired()])
    text = TextAreaField('Text', validators=[DataRequired()])
    submit = SubmitField('Post')
```

We have sucessfully created forms structure.

Now we create api ends for authentication
End points are 1. Register 2. Login 3. Logout

Create one api then show the result, and then repeat

```python
# user views
@users_posts.route('/register', methods=['GET', 'POST'])
def register():
  form = RegistrationForm()

  if form.validate_on_submit():
    user = User(email = form.email.data, password=form.password.data, username = form.username.data)
    db.session.add(user)
    db.session.commit()
    flash('Thanks for registration!')
    return redirect(url_for('users_posts.login'))

  return render_template('register.html', form=form)
```

In order to view register page you will have to update the navbar in `base.html`

Replace the navbar register link code

```html
<li class="nav-item">
  <a class="nav-link" href="{{url_for('users_posts.register')}}">Register</a>
</li>
```

```python
@users_posts.route('/login', methods=['GET', 'POST'])
def login():
  form = LoginForm()

  if form.validate_on_submit():
    user = User.query.filter_by(email=form.email.data).first()
    if user.check_password(form.password.data) and user is not None:
      login_user(user)
      flash('Log in successfully!')

      next = request.args.get('next')
      if next == None or not next[0] == '/':
        next = url_for('core.index')

      return redirect(next)

  return render_template('login.html', form=form)
```

```html
<li class="nav-item">
  <a class="nav-link" href="{{url_for('users_posts.login')}}">Login</a>
</li>
```

```python
@users_posts.route('/logout')
def logout():
  logout_user()
  return redirect(url_for('core.index'))
```

```html
<li class="nav-item">
  <a class="nav-link" href="{{url_for('users_posts.logout')}}">Logout</a>
</li>
```

------- Done with auth endpoints --------

Add account and username endpoints

```python
@users_posts.route('/account', methods=['GET', 'POST'])
@login_required
def account():
  username = current_user.username
  email = current_user.email
  display_picture = f'https://ui-avatars.com/api/?name={username[0]}'
  return render_template('account.html', username=username, email=email, display_picture=display_picture)

```

```html
<li class="nav-item">
  <a class="nav-link" href="{{url_for('users_posts.account')}}">Account</a>
</li>
```

```python
@users_posts.route('/<username>')
def user_posts(username):
  page = request.args.get('page', 1, type=int)
  user = User.query.filter_by(username=username).first_or_404()
  blog_posts = BlogPost.query.filter_by(author=user).order_by(
      BlogPost.date.desc()).paginate(page=page, per_page=5)
  display_picture = f'https://ui-avatars.com/api/?name={username[0]}'
  return render_template('user_blog_posts.html', blog_posts=blog_posts, user=user, display_picture=display_picture)
```

---- Blog related endpoints -----

Now we work with blog related endpoints
such as

1. Creating blogs
2. Showing Blogs
3. Deleting Blogs

4. Creating blog (/blogify/views.py)

```python
# blog views
@users_posts.route('/create', methods=['GET', 'POST'])
@login_required
def create_post():
  form  = BlogPostForm()

  if form.validate_on_submit():
    blog_post = BlogPost(title = form.title.data, text = form.text.data, user_id = current_user.id)
    db.session.add(blog_post)
    db.session.commit()
    flash('Blog Post Created')
    return redirect(url_for('core.index'))

  return render_template('create_post.html', form = form)
```

```html
<li class="nav-item">
  <a class="nav-link" href="{{url_for('users_posts.create_post')}}"
    >Create Post</a
  >
</li>
```

2. Display Blog

We change code in core views file
Replace the code for index route

```python
@core.route('/')
def index():
    page = request.args.get('page', 1, type=int)
    blog_posts = BlogPost.query.order_by(BlogPost.date.desc()).paginate(page = page, per_page=5)

    return render_template('index.html', blog_posts= blog_posts)
```

And now uncomment the code in `index.html` so that blogs would render.

Now we check the index endpoint

3. Now We create dynamic endpoint to view individual page

```python
@users_posts.route('/<int:blog_post_id>')
def blog_post(blog_post_id):
  blog_post = BlogPost.query.get_or_404(blog_post_id)
  return render_template('blog_post.html', post = blog_post)
```

4. Endpoint for deleting a blog

```python
@users_posts.route('/<int:blog_post_id>/delete', methods=['GET', 'POST'])
@login_required
def delete_post(blog_post_id):
  blog_post = BlogPost.query.get_or_404(blog_post_id)
  if blog_post.author != current_user:
    abort(403)

  db.session.delete(blog_post)
  db.session.commit()
  flash('Blog Post Deleted')
  return redirect(url_for('core.index'))
```

------ Done with blogs -------

Lets replace the navbar to make it dynamic 

> You will need to uncomment the body code and comment the existing navbar code in `base.html`

----- Done ----

> Hurray we completed the project!!!

