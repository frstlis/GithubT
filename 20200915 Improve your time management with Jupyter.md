[#]: collector: (lujun9972)
[#]: translator: (stevenzdg988)
[#]: reviewer: ( )
[#]: publisher: ( )
[#]: url: ( )
[#]: subject: (Improve your time management with Jupyter)
[#]: via: (https://opensource.com/article/20/9/calendar-jupyter)
[#]: author: (Moshe Zadka https://opensource.com/users/moshez)

Improve your time management with Jupyter
使用 Jupyter 改善你的时间管理
======
Discover how you are spending time by parsing your calendar with Python
in Jupyter.
![Calendar close up snapshot][1]
通过在 Jupyter 里使用 Python 分析日历了解您如何使用时间 。
![日历特写快照][1]

[Python][2] has incredibly scalable options for exploring data. With [Pandas][3] or [Dask][4], you can scale [Jupyter][5] up to big data. But what about small data? Personal data? Private data?
[Python][2] 在用于探索数据方面具有令人难以置信的可扩展性选项。利用 [Pandas][3] 或 [Dask][4]，您可以将 [Jupyter][5] 扩展到大数据(处理)。但是小数据呢？个人资料？私人数据？

JupyterLab and Jupyter Notebook provide a great environment to scrutinize my laptop-based life.
JupyterLab 和 Jupyter Notebook 提供了一个绝佳的环境来仔细检查我基于笔记本电脑的生活。

My exploration is powered by the fact that almost every service I use has a web application programming interface (API). I use many such services: a to-do list, a time tracker, a habit tracker, and more. But there is one that almost everyone uses: _a calendar_. The same ideas can be applied to other services, but calendars have one cool feature: an open standard that almost all web calendars support: `CalDAV`.
我的探索是基于以下事实：我使用的几乎每个服务都有一个 Web 应用程序编程接口（API）。我使用了许多此类服务：待办事项列表，时间跟踪器，习惯跟踪器等。但是几乎每个人都使用：_日历_。相同的想法可以应用于其他服务，但是日历具有一个很酷的功能：几乎所有网络日历都支持的开放标准： `CalDAV`。

### Parsing your calendar with Python in Jupyter
在 Jupyter 中使用 Python 分析日历

Most calendars provide a way to export into the `CalDAV` format. You may need some authentication for accessing this private data. Following your service's instructions should do the trick. How you get the credentials depends on your service, but eventually, you should be able to store them in a file. I store mine in my root directory in a file called `.caldav`:
大多数日历提供了导出为 `CalDAV` 格式的方法。您可能需要某种身份验证才能访问此私有数据。按照您的服务说明进行操作即可。如何获得凭据取决于您的服务，但是最终，您应该能够将它们存储在文件中。我将我的名为 `.caldav` 的文件存储在根(root)目录中：

```
import os
with open(os.path.expanduser("~/.caldav")) as fpin:
    username, password = fpin.read().split()
```

Never put usernames and passwords directly in notebooks! They could easily leak with a stray `git push`.
切勿将用户名和密码直接放在笔记本中！他们可能会很容易因 `git push` 偏离而导致泄漏。

The next step is to use the convenient PyPI [caldav][6] library. I looked up the CalDAV server for my email service (yours may be different):
下一步是使用方便的 PyPI [caldav][6] 库。我在 CalDAV 服务器上查找了我的电子邮件服务（您可能有所不同）：

```
import caldav
client = caldav.DAVClient(url="<https://caldav.fastmail.com/dav/>", username=username, password=password)
```

CalDAV has a concept called the `principal`. It is not important to get into right now, except to know it's the thing you use to access the calendars:
CalDAV 有一个称为 `principal`(主键) 的概念。马上进入(该系统)并不重要，除非这是您用于访问日历：

```
principal = client.principal()
calendars = principal.calendars()
```

Calendars are, literally, all about time. Before accessing events, you need to decide on a time range. One week should be a good default:
日历按照字面讲是关于时间的。访问事件之前，您需要确定一个时间范围。默认设置为一星期：

```
from dateutil import tz
import datetime
now = datetime.datetime.now(tz.tzutc())
since = now - datetime.timedelta(days=7)
```

Most people use more than one calendar, and most people want all their events together. The `itertools.chain.from_iterable` makes this straightforward: ` `
大多数人使用不止一个日历，并且希望所有事件都在一起。`itertools.chain.from_iterable` (方法)使这一过程变得简单：` `

```
import itertools

raw_events = list(
    itertools.chain.from_iterable(
        calendar.date_search(start=since, end=now, expand=True)
        for calendar in calendars
    )
)
```

Reading all the events into memory is important, and doing so in the API's raw, native format is an important practice. This means that when fine-tuning the parsing, analyzing, and displaying code, there is no need to go back to the API service to refresh the data.
将所有事件读入内存很重要，以 API 原始的本地格式进行操作是重要的做法。这意味着在调整解析，分析和显示代码时，无需返回到 API 服务刷新数据。

But "raw" is not an understatement. The events come through as strings in a specific format:
但 “原始” 并不是轻描淡写。事件通过特定格式的字符串实现：

```
`print(raw_events[12].data)`[/code] [code]

    BEGIN:VCALENDAR
    VERSION:2.0
    PRODID:-//CyrusIMAP.org/Cyrus
     3.3.0-232-g4bdb081-fm-20200825.002-g4bdb081a//EN
    BEGIN:VEVENT
    DTEND:20200825T230000Z
    DTSTAMP:20200825T181915Z
    DTSTART:20200825T220000Z
    SUMMARY:Busy
    UID:
     1302728i-040000008200E00074C5B7101A82E00800000000D939773EA578D601000000000
     000000010000000CD71CC3393651B419E9458134FE840F5
    END:VEVENT
    END:VCALENDAR
```

Luckily, PyPI comes to the rescue again with another helper library, [vobject][7]:
幸运的是，PyPI 再次使用另一个帮助程序库 [vobject][7] 前来解围：

```
import io
import vobject

def parse_event(raw_event):
    data = raw_event.data
    parsed = vobject.readOne(io.StringIO(data))
    contents = parsed.vevent.contents
    return contents

[/code] [code]`parse_event(raw_events[12])`[/code] [code]

    {'dtend': [&lt;DTEND{}2020-08-25 23:00:00+00:00&gt;],
     'dtstamp': [&lt;DTSTAMP{}2020-08-25 18:19:15+00:00&gt;],
     'dtstart': [&lt;DTSTART{}2020-08-25 22:00:00+00:00&gt;],
     'summary': [&lt;SUMMARY{}Busy&gt;],
     'uid': [&lt;UID{}1302728i-040000008200E00074C5B7101A82E00800000000D939773EA578D601000000000000000010000000CD71CC3393651B419E9458134FE840F5&gt;]}
```

Well, at least it's a little better.
好吧，至少好一点了。

There is still some work to do to convert it to a reasonable Python object. The first step is to _have_ a reasonable Python object. The [attrs][8] library provides a nice start:
仍有一些工作要做，以将其转换为合理的 Python 对象。第一步是 _拥有_ 一个合理的 Python 对象。[attrs][8] 库提供了一个不错的开始：

```
import attr
from __future__ import annotations
@attr.s(auto_attribs=True, frozen=True)
class Event:
    start: datetime.datetime
    end: datetime.datetime
    timezone: Any
    summary: str
```

Time to write the conversion code!

The first abstraction gets the value from the parsed dictionary without all the decorations:
是时候编写转换代码了！

第一个抽象在没有所有修饰的情况下从解析字典中获取值：

```
def get_piece(contents, name):
    return contents[name][0].value

[/code] [code]`get_piece(_, "dtstart")`[/code] [code]`    datetime.datetime(2020, 8, 25, 22, 0, tzinfo=tzutc())`
```

Calendar events always have a start, but they sometimes have an "end" and sometimes a "duration." Some careful parsing logic can harmonize both into the same Python objects:
日历事件总是有一个开始，有一个“结束”，有一个 “持续时间”。一些谨慎的解析逻辑可以将两者协调为同一个 Python 对象：

```
def from_calendar_event_and_timezone(event, timezone):
    contents = parse_event(event)
    start = get_piece(contents, "dtstart")
    summary = get_piece(contents, "summary")
    try:
        end = get_piece(contents, "dtend")
    except KeyError:
        end = start + get_piece(contents, "duration")
    return Event(start=start, end=end, summary=summary, timezone=timezone)
```

Since it is useful to have the events in your _local_ time zone rather than UTC, this uses the local timezone:
由于将事件放在 _本地_ 时区而不是 UTC 中很有用，因此使用本地时区：

```
`my_timezone = tz.gettz()`[/code] [code]`from_calendar_event_and_timezone(raw_events[12], my_timezone)`[/code] [code]`    Event(start=datetime.datetime(2020, 8, 25, 22, 0, tzinfo=tzutc()), end=datetime.datetime(2020, 8, 25, 23, 0, tzinfo=tzutc()), timezone=tzfile('/etc/localtime'), summary='Busy')`
```

Now that the events are real Python objects, they really should have some additional information. Luckily, it is possible to add methods retroactively to classes.
既然事件是真实的Python对象，那么它们实际上应该具有附加信息。 幸运的是，将方法添加到类中是可能的。

But figuring which _day_ an event happens is not that obvious. You need the day in the _local_ timezone:
但是弄清楚哪个事件发生在哪一天不是很明显。您需要在 _本地_ 时区中选择一天：

```
def day(self):
    offset = self.timezone.utcoffset(self.start)
    fixed = self.start + offset
    return fixed.date()
Event.day = property(day)

[/code] [code]`print(_.day)`[/code] [code]`    2020-08-25`
```

Events are always represented internally as start/end, but knowing the duration is a useful property. Duration can also be added to the existing class:
事件始终在内部表示为开始/结束，但是知道持续时间是有用的属性。持续时间也可以添加到现有类中：

```
def duration(self):
    return self.end - self.start
Event.duration = property(duration)

[/code] [code]`print(_.duration)`[/code] [code]`    1:00:00`
```

Now it is time to convert all events into useful Python objects:
现在到了将所有事件转换为有用的 Python 对象了：

```
all_events = [from_calendar_event_and_timezone(raw_event, my_timezone)
              for raw_event in raw_events]
```

All-day events are a special case and probably less useful for analyzing life. For now, you can ignore them:
全天事件是一种特例，可能对分析生活没有多大用处。现在，您可以忽略它们：

```
# ignore all-day events
all_events = [event for event in all_events if not type(event.start) == datetime.date]
```

Events have a natural order—knowing which one happened first is probably useful for analysis:
事件具有自然顺序——知道哪个事件最先发生可能有助于分析：

```
`all_events.sort(key=lambda ev: ev.start)`
```

Now that the events are sorted, they can be broken into days:
现在，事件已排序，可以将它们加载到每天：

```
import collections
events_by_day = collections.defaultdict(list)
for event in all_events:
    events_by_day[event.day].append(event)
```

And with that, you have calendar events with dates, duration, and sequence as Python objects.
这样，您就可以将带有日期，持续时间和序列的日历事件作为 Python 对象。
### Reporting on your life in Python
用 Python 报到你的生活

Now it is time to write reporting code! It is fun to have eye-popping formatting with proper headers, lists, important things in bold, etc.
现在是时候编写报告代码了！带有适当的标题，列表，重要内容以粗体显示令人惊奇的格式是很有趣的。

This means HTML and some HTML templating. I like to use [Chameleon][9]:
这就是 HTML 和 HTML 模板。我喜欢使用 [Chameleon(变色龙：动态的)][9]：

```
template_content = """
&lt;html&gt;&lt;body&gt;
&lt;div tal:repeat="item items"&gt;
&lt;h2 tal:content="item[0]"&gt;Day&lt;/h2&gt;
&lt;ul&gt;
    &lt;li tal:repeat="event item[1]"&gt;&lt;span tal:replace="event"&gt;Thing&lt;/span&gt;&lt;/li&gt;
&lt;/ul&gt;
&lt;/div&gt;
&lt;/body&gt;&lt;/html&gt;"""
```

One cool feature of Chameleon is that it will render objects using its `html` method. I will use it in two ways:

  * The summary will be in **bold**
  * For most events, I will remove the summary (since this is my personal information)
Chameleon 的一个很酷的功能是使用 `html` 方法渲染对象。我将以两种方式使用它：

  * 摘要将以 **bold** (**粗体**)显示
  * 对于大多数活动，我都会删除摘要（因为这是我的个人信息）

```
def __html__(self):
    offset = my_timezone.utcoffset(self.start)
    fixed = self.start + offset
    start_str = str(fixed).split("+")[0]
    summary = self.summary
    if summary != "Busy":
        summary = "&amp;lt;REDACTED&amp;gt;"
    return f"&lt;b&gt;{summary[:30]}&lt;/b&gt; \-- {start_str} ({self.duration})"
Event.__html__ = __html__
```

In the interest of brevity, the report will be sliced into one day's worth.
为了简洁起见，该报告将切成每天的。

```
import chameleon
from IPython.display import HTML
template = chameleon.PageTemplate(template_content)
html = template(items=itertools.islice(events_by_day.items(), 3, 4))
HTML(html)
```

#### When rendered, it will look something like this:
渲染后，它将看起来像这样：

#### 2020-08-25

  * **&lt;编辑&gt;** \-- 2020-08-25 08:30:00 (0:45:00)
  * **&lt;编辑&gt;** \-- 2020-08-25 10:00:00 (1:00:00)
  * **&lt;编辑&gt;** \-- 2020-08-25 11:30:00 (0:30:00)
  * **&lt;编辑&gt;** \-- 2020-08-25 13:00:00 (0:25:00)
  * **Busy** \-- 2020-08-25 15:00:00 (1:00:00)
  * **&lt;编辑&gt;** \-- 2020-08-25 15:00:00 (1:00:00)
  * **&lt;编辑&gt;** \-- 2020-08-25 19:00:00 (1:00:00)
  * **&lt;编辑&gt;** \-- 2020-08-25 19:00:12 (1:00:00)



### Endless options with Python and Jupyter
Python 和 Jupyter 的无穷选择

This only scratches the surface of what you can do by parsing, analyzing, and reporting on the data that various web services have on you.
通过解析，分析和报告各种 Web 服务所拥有的数据，这只是您可以做的事情的表面。

Why not try it with your favorite service?
为什么不尝试使用您最喜欢的服务呢？

--------------------------------------------------------------------------------

via: https://opensource.com/article/20/9/calendar-jupyter

作者：[Moshe Zadka][a]
选题：[lujun9972][b]
译者：[stevenzdg988](https://github.com/stevenzdg988)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]: https://opensource.com/users/moshez
[b]: https://github.com/lujun9972
[1]: https://opensource.com/sites/default/files/styles/image-full-size/public/lead-images/calendar.jpg?itok=jEKbhvDT (Calendar close up snapshot)
[2]: https://opensource.com/resources/python
[3]: https://pandas.pydata.org/
[4]: https://dask.org/
[5]: https://jupyter.org/
[6]: https://pypi.org/project/caldav/
[7]: https://pypi.org/project/vobject/
[8]: https://opensource.com/article/19/5/python-attrs
[9]: https://chameleon.readthedocs.io/en/latest/
