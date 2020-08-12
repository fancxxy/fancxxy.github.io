---
title: 给comicd写的前端网站
date: 2018-06-13 19:30:51
tags:
- Python
- Flask
---

之前写了一个下载漫画的命令行工具comicd，确实可以轻松获取漫画资源了，但是并没有想象中的便捷。首先每次获取资源都需要执行下载命令，对于完结漫画还可以接受，一次下载一劳永逸，但对于长期连载的漫画就显得很不方便，其次不能随时随地在手机上查看，需要通过电脑下载传到手机上，所以在其基础上实现了一个web应用[ComicWeb](https://github.com/fancxxy/ComicWeb)，可以添加想要追的漫画，定时任务自动刷新资源。

<!--more-->

## 网站
ComicWeb是一个订阅和观看漫画的网站，支持主流的几个漫画资源网站，项目仅供个人学习，请勿用于商业盈利。项目中使用的主要框架和组件是Flask+Bootstrap+MySQL，后端的漫画爬取功能参阅[fancxxy/comicd](https://github.com/fancxxy/comicd)。

{% asset_img comic-list.jpg %}
{% asset_img chapter-list.jpg %}
{% asset_img comic-picture.jpg %}

## 程序
ComicWeb的后端功能是用Flask实现的，Flask是一个用Python编写的轻量级Web框架，扩展能力强，程序的工程目录大致如下，requirements.txt记录所有依赖的扩展，config.py是工程的配置文件，在初始化Flask程序实例的时候通过参数传递过去，web.py是启动脚本，在环境变量中设置启动脚本的值`FLASK_APP=web.py`，app/templates是模板文件的存放目录，app/static存放静态文件，app/main是蓝本目录，程序的主要路由功能都定义在该蓝本目录下，app/models.py定义数据库表结构，app/auth是登录认证的蓝本，assist包含定时更新任务。

```
|-ComicWeb
  |-app/
    |-templates/
    |-static/
    |-main/
      |-__init__.py
      |-errors.py
      |-forms.py
      |-views.py
    |-auth/
      |-__init__.py
      |-forms.py
      |-views.py
    |-__init__.py
    |-models.py
  |-assist
    |-__init__.py
    |-update.py
  |-requirements.txt
  |-config.py
  |-web.py 
```

### 主表结构
Flask程序一般都配合数据库抽象框架SQLAlchemy一起使用，可以使用面向对象的操作来处理数据库指令。首先需要安装Flask得扩展`flask-sqlalchemy`，包装了SQLAlchemy的操作，对于MySQL的连接串格式是`mysql://username:password@hostname/database`，同时表结构为了获得抽象能力，定义时需要继承`db.Model`类，使用`__tablename__`来确定表名，表字段通过`db.Column`创建。

``` python
class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(64), unique=True, index=True)
    username = db.Column(db.String(64), unique=True, index=True)
    password_hash = db.Column(db.String(128))
```

ComicWeb网站的两个主要的对象分别是用户和漫画，用户登录后可以订阅多个漫画资源，某个漫画资源同时也可以被多个用户订阅，所以对于用户和漫画的关系是Many to Many，根据SQLAlchemy的文档Many to Many的关系需要在两个类之间建立一个关联表来体现订阅关系，关联表不需要抽象，直接用`db.table`创建。

``` python
subscribers = db.Table('subscribers',
                        db.Column('user_id', db.Integer, db.ForeignKey('users.id'), primary_key=True),
                        db.Column('comic_id', db.Integer, db.ForeignKey('comics.id'), primary_key=True)
```

这种创建方式的表不能被抽象，意味着不能从`subscribers`表获取额外有用信息，如果想要存储订阅时间就需要抽象`subscribers`表，这里额外记录了用户对于某个漫画的订阅日期和该漫画的观看进度。关系的两端`User`和`Comic`分别通过`db.relationship`来指定对端关系，方法的第一额参数是关系对端的模型名字，`backref`参数用来在对端模型中创建一个反向引用，`lazy='dynamic'`表明`comcis`的加载方式是调用`query`方法时加载记录，`cascade`参数的值由逗号分隔，把对象添加到会话后会把关联的所有对象都添加到会话，同时删除对象时会把关联的所有孤儿记录都删除掉。

``` python
class Subscriber(db.Model):
    __tablename__ = 'subscribers'
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'), primary_key=True)
    comic_id = db.Column(db.Integer, db.ForeignKey('comics.id'), primary_key=True)
    subscribe_date = db.Column(db.DateTime(), default=datetime.utcnow)
    last_chapter_id = db.Column(db.Integer)
    
class User(db.Model):
    # ...
    comics = db.relationship('Subscriber', backref=db.backref('user', lazy='joined'), lazy='dynamic',
                             cascade='all, delete-orphan')

class Comic(db.Model):
    __tablename__ = 'comics'
    id = db.Column(db.Integer, primary_key=True)
    url = db.Column(db.String(256), unique=True, nullable=False)
    title = db.Column(db.String(128), nullable=False)
    interface = db.Column(db.String(32), nullable=False)
    cover = db.Column(db.String(256))
    summary = db.Column(db.Text())
    newest_chapter_id = db.Column(db.Integer)
    newest_chapter_title = db.Column(db.String(128))
    update_time = db.Column(db.DateTime(), default=None)

    users = db.relationship('Subscriber', backref=db.backref('comic', lazy='joined'), lazy='dynamic',
                            cascade='all, delete-orphan')
```

有了上述关系模型就可以很方便的查询某个用户订阅的漫画列表，以及某个漫画的所有订阅用户。

``` Python
>>> user = User.query.filter_by(id=1).first()
>>> user
<User fxx>
>>> [item.comic for item in user.comics]
[<Comic 航海王 腾讯漫画>, <Comic 银魂 腾讯漫画>, <Comic 死神/境·界 腾讯漫画>, <Comic 火影忍者 腾讯漫画>, <Comic 龙珠 腾讯漫画>]
>>>
>>> comic = Comic.query.filter_by(id=1).first()
>>> comic
<Comic 航海王 腾讯漫画>
>>> [item.user for item in comic.users]
[<User fxx>, <User fancxxy>]

```

### 登录认证
登录以及管理会话的功能需要用到的扩展列表有

* flask-login: 管理已登录的用户会话
* werkzeug: 计算密码散列值并在每次输入密码时进行核对
* itsdangerous: 生成核对加密令牌

创建登录页面的表单，需要安装扩展`flask_wtf`，表单类都继承自`FlaskForm`类，表单中的项都是wtforms模块中的类实例，`validatiors`参数是一个验证函数组成的列表，`render_kw`是一个属性字典，在渲染`input`标签时添加`placeholder="输入电子邮件"`属性。

``` python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, BooleanField, SubmitField
from wtforms.validators import DataRequired, Length, Email


class LoginForm(FlaskForm):
    email = StringField('邮件地址:', validators=[DataRequired(), Length(1, 64), Email()], render_kw={'placeholder': '输入邮件地址'})
    password = PasswordField('密码:', validators=[DataRequired()], render_kw={'placeholder': '输入密码'})
    remember_me = BooleanField('保持登录')
    submit = SubmitField('登录')
```

密码肯定不能明文存储，也不让直接读取，使用`@property`装饰物改变`password`变量的读写属性，使用`Werkzeug`的`security`模块来处理密码散列值，`generate_password_hash`方法计算输入密码的散列值，`check_password_hash`方法比较输入的密码和存储在数据库中的散列值是否匹配，`flask-login`提供了多个方法校验用户是否登录，是否激活，本地用户模型想要使用这些方法就需要继承`UserMixin`类。

``` python
class User(db.Model, UserMixin):
    # ...
    @property
    def password(self):
        raise AttributeError('password is not a readable attribute')

    @password.setter
    def password(self, password):
        self.password_hash = generate_password_hash(password)

    def verify_password(self, password):
        return check_password_hash(self.password_hash, password)

```

定义登录和登出的视图函数，一旦密码验证成功后调用`flask-login`的`login_user`方法在会话中把用户标记为已登录，`logout_user`方法让用户登出。

``` python
@auth.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        user = User.query.filter_by(email=form.email.data).first()
        if user is not None and user.verify_password(form.password.data):
            login_user(user, form.remember_me.data)
            return redirect(request.args.get('next') or url_for('main.index'))
        flash('错误的邮件地址或者密码', 'warning')
    return render_template('login.html', form=form)


@auth.route('/logout')
@login_required
def logout():
    logout_user()
    flash('您已经退出', 'info')
    return redirect(url_for('main.index'))
```

前端框架Bootstrap通过`flask-bootstrap`扩展可以很方便的和Flask的模版系统集成在一起，该扩展提供`quick_form`宏方法渲染表单对象。

``` html
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Comic - Login{% endblock %}

{% block page_content %}
    <div class="col-lg-12">
        <div class="page-header">
            <h1>登录</h1>
        </div>
        <div class="col-sm-4">
            {{ wtf.quick_form(form) }}
        </div>
    </div>
{% endblock %}
```

登录页面截图

### 功能路由
主页面有一个订阅新漫画的输入框，下面列出该用户订阅的所有漫画列表，每个漫画资源都展示了标题、封面、更新时间和来源。
让用户订阅漫画的逻辑其实就是向`subscribers`表插入一条数据，在`User`模型中添加`subscribe`方法实现这个功能。

``` python
class User(db.Model, UserMixin):
    # ...
    def subscribe(self, comic):
        if self.comics.filter_by(comic_id=comic.id).first() is None:
        s = Subscriber(user=self, comic=comic)
        db.session.add(s)
```

不同网站提供的漫画封面大小不一样，为了风格统一，写了一个方法裁剪封面大小，把高度和宽度的比例设置成了4:3，需要用到Python的图像处理库`PIL`。

``` python
from PIL import Image

def crop_cover(filename):
    with Image.open(filename) as im:
        width, height = im.size

        # width divide height must be equal 3/4
        if width / height > 0.75:
            w = width - height * 0.75
            im.crop((w // 2, 0, width - w // 2, height)).save(filename)
            im.crop((0, 10, 300, 400)).save()
        elif width / height < 0.75:
            h = height - width // 0.75
            im.crop((0, h // 2, width, height - h // 2)).save(filename)
```

章节作为资源对象也需要被抽象化，`Comic`跟`Chapter`的关系是one to many，需要在`Comic`模型中指明关系。

``` python
class Chapter(db.Model):
    __tablename__ = 'chapters'
    id = db.Column(db.Integer, primary_key=True)
    url = db.Column(db.String(256), unique=True, nullable=False)
    chapter_no = db.Column(db.Integer)
    title = db.Column(db.String(128))
    interface = db.Column(db.String(32))
    comic_title = db.Column(db.String(128))
    comic_id = db.Column(db.Integer, db.ForeignKey('comics.id'), nullable=False)
    update_time = db.Column(db.DateTime(), default=datetime.utcnow)
    path = db.Column(db.String(256), default=None)
    __table_args__ = (db.UniqueConstraint('title', 'comic_title', 'interface', name='uc_title_comic_title_interface'),
                      db.UniqueConstraint('chapter_no', 'comic_id', name='uc_no_id'))

    def __repr__(self):
        return '<Chapter {} {} {}>'.format(self.comic_title, self.title, self.interface)


class Comic(db.Model):
    # ...
    chapters = db.relationship('Chapter', backref='comic', lazy='dynamic')
```

订阅漫画表单需要校验漫画来源是否支持，`ComicWeb`获取漫画资源使用的是`comicd`的接口，`comicd`提供的`Comic.find`静态方法可以校验提交的url地址是否支持，不支持会抛出`ComicdError`的异常。

``` python
from comicd import Comic, ComicdError

class SubscribeForm(FlaskForm):
    url = StringField('Comic URL:', validators=[DataRequired(), URL()], render_kw={'placeholder': '输入漫画网址'})
    submit = SubmitField('订阅')

    def validate_url(self, field):
        try:
            Comic.find(field.data)
        except ComicdError:
            raise ValidationError('只支持 {}'.format(' '.join(list(Comic.support()))))
```

主页面视图函数的逻辑是这样的，首先验证一下提交的表单数据，如果提交的漫画数据已经存在在数据库中，则直接让用户订阅该漫画，如果漫画还没收录就需要使用`comicd`提供的接口去对应网站请求漫画数据，向`comics`表插入漫画数据，下载漫画封面，让用户订阅漫画，向`chapters`表插入所有章节的数据，业务处理完成后重定向到主页面，通过分页查询返回数据，最后渲染页面。

``` python
@main.route('/', methods=['GET', 'POST'])
@login_required
def index():
    form = SubscribeForm()
    if form.validate_on_submit():
        comic = Comic.query.filter_by(url=form.url.data).first()
        if comic:
            current_user.subscribe(comic)
            flash('订阅成功', 'success')
            return redirect(url_for('main.index'))
        else:
            c = cc(form.url.data)
            if c.init():
                hash = md5((c.instance.name + c.title).encode('utf-8')).hexdigest()
                comic = Comic(url=c.url, title=c.title, interface=c.instance.name, cover=hash + '.jpg',
                              summary=c.summary)
                r, f = c.download_cover('app/static/covers', comic.cover)
                if r:
                    crop_cover(f)
                current_user.subscribe(comic)
                db.session.add(comic)
                db.session.commit()
                index = 1
                for title, url in c.chapters:
                    chapter = Chapter(title=title, comic_title=comic.title, interface=comic.interface, url=url,
                                      comic_id=comic.id, chapter_no=index)
                    db.session.add(chapter)
                    index += 1
                db.session.commit()
                chapter = Chapter.query.filter_by(comic_id=comic.id).order_by(db.desc(Chapter.id)).first()
                comic.newest_chapter_id, comic.newest_chapter_title = chapter.id, chapter.title
                db.session.add(comic)
                db.session.commit()
                flash('订阅成功', 'success')
                return redirect(url_for('main.index'))
            else:
                flash('订阅失败', 'danger')
    page = request.args.get('page', 1, type=int)
    pagination = current_user.comics.paginate(page, per_page=current_app.config['COMICS_PER_PAGE'], error_out=False)
    comics = [item.comic for item in pagination.items]
    return render_template('index.html', comics=comics, pagination=pagination, form=form)
```

表单框不能使用`form.quick_form()`直接渲染，手写一个输入框，`form.hidden_tag()`是为了隐藏CSRF信息。

``` html
<form method="POST" class="form-form">
    {{ form.hidden_tag() }}
    <div class="input-group">
        {{ form.url(class="form-control input-sm") }}
        <span class="input-group-btn">
            {{ form.submit(class="btn btn-primary btn-sm") }}
        </span>
    </div>
</form>
```

添加订阅的逻辑是向`subscribers`表插入数据，那么取消订阅的逻辑就是删除这条数据。

``` python
@main.route('/unsubscribe/<id>')
@login_required
def unsubscribe(id):
    subscriber = Subscriber.query.filter_by(user_id=current_user.id, comic_id=id).first()
    if subscriber:
        db.session.delete(subscriber)
    flash('取消订阅', 'info')
    return redirect(url_for('main.index'))
```

取消按钮放在封面图的左下角，当鼠标移动过去背景设置为黑色半透明，移开之后背景设置为透明，`:hover`选择器可以控制鼠标指针浮动的效果。

``` css
.shownn .unsubscribe {
    /* */
    background: transparent;
}

.shownn:hover .unsubscribe {
    /* */
    background: rgba(0, 0, 0, .50);
}
```

``` html
<div class="shownn"><a class="unsubscribe" href="{{ url_for('main.unsubscribe', id=comic.id) }}" title="取消订阅">取消订阅</a></div>
```

漫画主页的路由是`/comics/<id>`，这个页面展示了更详细的漫画信息、观看进度以及章节标题，观看进度保存在了`subscribers`表中，需要用`user_id`和`comic_id`去表里查询数据，`pagination(page, per_page=20, error_out=False)`方法由`flask-sqlalchemy`提供，方法返回一个`Pagination`分页对象。

``` python
@main.route('/comics/<id>')
@login_required
def comic(id):
    comic = Comic.query.get_or_404(id)
    page = request.args.get('page', 1, type=int)
    pagination = comic.chapters.order_by(db.desc(Chapter.id)).paginate(page,
                                                                       per_page=current_app.config['CHAPTERS_PER_PAGE'],
                                                                       error_out=False)
    chapters = pagination.items
    last_chapter = Chapter.query.filter_by(id=Subscriber.query.filter_by(user_id=current_user.id,
                                                                         comic_id=id).first().last_chapter_id).first()
    newest_chapter = Chapter.query.filter_by(id=comic.newest_chapter_id).first()
    return render_template('comic.html', comic=comic, chapters=chapters, pagination=pagination,
                           last_chapter=last_chapter, newest_chapter=newest_chapter)
```

`Pagination`分页对象提供`iter_pages`方法返回一个迭代器，存储的是分页导航显示的页数列表，在flasky的示例代码中就用这个迭代器创建了一个分页导航，这里我直接拿来用了。

``` html
{% macro pagination_widget(pagination, endpoint) %}
    <ul class="pagination">
        <li{% if not pagination.has_prev %} class="disabled"{% endif %}>
            <a href="{% if pagination.has_prev %}{{ url_for(endpoint, page=pagination.prev_num, **kwargs) }}{% else %}#{% endif %}">
                &laquo;
            </a>
        </li>
        {% for p in pagination.iter_pages(left_edge=1, left_current=1, right_current=3, right_edge=1) %}
            {% if p %}
                {% if p == pagination.page %}
                    <li class="active">
                        <a href="{{ url_for(endpoint, page = p, **kwargs) }}">{{ p }}</a>
                    </li>
                {% else %}
                    <li>
                        <a href="{{ url_for(endpoint, page = p, **kwargs) }}">{{ p }}</a>
                    </li>
                {% endif %}
            {% else %}
                <li class="disabled"><a href="#">&hellip;</a></li>
            {% endif %}
        {% endfor %}
        <li{% if not pagination.has_next %} class="disabled"{% endif %}>
            <a href="{% if pagination.has_next %}{{ url_for(endpoint, page=pagination.next_num, **kwargs) }}{% else %}#{% endif %}">
                &raquo;
            </a>
        </li>
    </ul>
{% endmacro %}
```

图片也是资源，每张图片都有一条记录在`images`表里，以`comic_id`和`chapter_id`作为外键。

``` python
class Image(db.Model):
    __tablename__ = 'images'
    id = db.Column(db.Integer, primary_key=True)
    comic_id = db.Column(db.Integer, db.ForeignKey('comics.id'), nullable=False)
    chapter_id = db.Column(db.Integer, db.ForeignKey('chapters.id'), nullable=False)
    image_no = db.Column(db.Integer, nullable=False)
    path = db.Column(db.String(256), nullable=False)
```

图片资源的下载逻辑是这样的，有用户第一次请求该章节数据，去对应网站下载图片资源，把每张图片编号，编号和路径存储到`images`表中，之后再有用户请求该章节直接从`images`表给出查询结果。

``` python
@main.route('/comics/<id>/<no>')
@login_required
def chapter(id, no):
    chapter = Chapter.query.filter_by(comic_id=id, chapter_no=no).first()
    if not chapter.path:
        c = cr(chapter.url)
        if c.init():
            c.download()
        count, chapter.path = 1, join(cg.home, quote(c.title), quote(c.ctitle))
        if exists(chapter.path):
            files = sorted([i for i in listdir(chapter.path) if splitext(i)[1] == '.jpg'], key=lambda x: int(x[:-4]))
            for file in files:
                image = Image(comic_id=id, chapter_id=chapter.id, image_no=count,
                              path=join(quote(c.title), quote(c.ctitle), file))
                db.session.add(image)
                count += 1

    subscriber = Subscriber.query.filter_by(user_id=current_user.id, comic_id=chapter.comic_id).first()
    subscriber.last_chapter_id = chapter.id
    db.session.add(subscriber)
    images = Image.query.filter_by(comic_id=id, chapter_id=chapter.id).order_by(Image.image_no).all()
    return render_template('chapter.html', chapter=chapter, images=images, previous=request.referrer)
```

### 定时任务
长期连载的漫画会不定期更新，需要写一个方法来更新漫画章节目录，通过`comicd`的接口请求当前漫画的章节列表，判断列表末尾的数据和当前数据库的最新章节是否相同，如果不同说明有新章节更新了，在请求到的章节列表中找到当前数据库标记的最新章节数据的索引，索引后面的章节信息就是本次需要插到数据库的数据。

``` python
class Comic(db.Model):
    # ...
    def update(self):
        c = cc(self.url)
        if c.init():
            chapter = Chapter.query.filter_by(id=self.newest_chapter_id).first()
            if chapter and chapter.title != c.chapters[-1][0]:
                new_chapters = c.chapters[c.chapters.index((chapter.title, chapter.url)) + 1:]
                no = chapter.chapter_no + 1
                for title, url in new_chapters:
                    chapter = Chapter(title=title, comic_title=self.title, interface=self.interface, url=url,
                                      comic_id=self.id, chapter_no=no)
                    db.session.add(chapter)
                    no += 1
                new_chapter = Chapter.query.filter_by(comic_id=self.id).order_by(db.desc(Chapter.id)).first()
                self.newest_chapter_id, self.newest_chapter_title = new_chapter.id, new_chapter.title
                self.update_time = datetime.utcnow()
                db.session.add(self)
                return 'comic {} update at {}'.format(self.title, datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S'))
```

在assist/update.py文件中添加更新的方法。

``` python
def update_comics():
    with db.app.app_context():
        [comic.update() for comic in Comic.query.all()]
```

定时任务通过`flask-apscheduler`扩展实现，设置每隔一段时间调用上面的更新方法。

``` python
from assist.update import update_comics
    cron.add_job(id='update_comics', func=update_comics, trigger='interval', hours=app.config['UPDATE_HOURS'])
    cron.start()
```

## 部署
最后一步是部署，一般使用的web服务器是uwsgi，通过uWSGI协议和Flask程序交换数据，外部使用nginx转发请求。

uwsgi的配置

```
[uwsgi]

# uwsgi 启动时所使用的地址与端口
socket = 127.0.0.1:8001 

# 指向网站目录
chdir = /home/fancxxy/Works/ComicWeb

# python 启动程序文件
wsgi-file = /home/fancxxy/Works/ComicWeb/web.py

# python 程序内用以启动的 application 变量名
callable = app

# 处理器数
processes = 4

# 线程数
threads = 2

#状态检测地址
stats = 127.0.0.1:9191
```

nginx的配置

```
location / {
    include uwsgi_params;
    uwsgi_pass 127.0.0.1:8001;
    uwsgi_param UWSGI_PYTHON /home/fancxxy/Works/ComicWeb/venv;
    uwsgi_param UWSGI_CHDIR /home/fancxxy/Works/ComicWeb;
    uwsgi_param UWSGI_SCRIPT web:app;
}
```

uwsgi通过systemd启动，给uwsgi写了一个启动脚本。

```
[Unit]
Description=uWSGI script by fancxxy
After=syslog.target

[Service]
ExecStart=/home/fancxxy/Works/ComicWeb/venv/bin/uwsgi --ini /home/fancxxy/Works/ComicWeb/uwsgi.ini
Restart=always
KillSignal=SIGQUIT
Type=notify
StandardError=syslog
NotifyAccess=all
EnvironmentFile=/home/fancxxy/Works/ComicWeb/env.sh

[Install]
WantedBy=multi-user.target
```

