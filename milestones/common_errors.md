## COMMON ERRORS AND FIXES

| Error | Fix |
|-------|-----|
| `ModuleNotFoundError: flask` | Run `pip install flask flask-sqlalchemy` |
| `OperationalError: no such table` | You didn't run `db.create_all()` inside app context |
| `KeyError: 'user_id'` | Session expired — add `@login_required` decorator |
| Template not found | Check spelling and that file is in `templates/` folder |
| `jinja2.UndefinedError` | Variable not passed to `render_template()` |
| Static files not loading | Use `url_for('static', filename='css/style.css')` |
| `IntegrityError` | Duplicate email — email must be unique |

---