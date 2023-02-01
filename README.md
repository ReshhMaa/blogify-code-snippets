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

