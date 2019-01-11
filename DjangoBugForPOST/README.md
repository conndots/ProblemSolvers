Django 1.3.7关于POST请求读取Body的Bug导致报错解决

=======



# 问题

公司的python项目还在使用比较老的版本django 1.3.7。我这里有一个Django Middleware，想要把每个请求的body(如果有)发送给自己的服务。  

最近，被反馈了一个报错：  

```python
#...
File "/xxx.py", line 103, in check_django_sign
    body = request.raw_post_data
  File "/xxx/django/http/__init__.py", line 267, in _get_raw_post_data
    raise Exception("You cannot access raw_post_data after reading from request's data stream")
Exception: You cannot access raw_post_data after reading from request's data stream
```

# 解析

搜索了一下，[StackOverflow](https://stackoverflow.com/questions/19581110/exception-you-cannot-access-body-after-reading-from-requests-data-stream)这里描述了类似的问题，主要是说，Django的文档里，直截了当建议不要在Middleware里使用request.POST。

> [M]iddleware that hits request.POST should (usually) be considered a bug. It means that the view will be unable to set any custom upload handlers, perform custom parsing of the request body, or enforce permission checks prior to file uploads being accepted.

大意就是，如果用了.POST，就无法添加一些custom的parser等了。但是，并没有说明为什么会出现这个exception。    

可是，线上，无论是在middleware里先调用POST，再调用_raw_post_data，还是反过来，都不会复现这个问题。

所以，就要阅读Django源码看看了。首先，我们看看抛出错误的地方的代码`http/__init__.py`，在类HttpRequest中：  

```python
    def read(self, *args, **kwargs):
        self._read_started = True #这里会设置_read_started
        return self._stream.read(*args, **kwargs)

    def readline(self, *args, **kwargs):
        self._read_started = True #这里会设置_read_started
        return self._stream.readline(*args, **kwargs)
    
    def _get_raw_post_data(self):
        if not hasattr(self, '_raw_post_data'):
            if self._read_started:
                raise Exception("You cannot access raw_post_data after reading from request's data stream") #这个地方抛出异常
            try:
                content_length = int(self.META.get('CONTENT_LENGTH', 0))
            except (ValueError, TypeError):
                # If CONTENT_LENGTH was empty string or not an integer, don't
                # error out. We've also seen None passed in here (against all
                # specs, but see ticket #8259), so we handle TypeError as well.
                content_length = 0
            if content_length:
                self._raw_post_data = self.read(content_length)
            else:
                self._raw_post_data = self.read()
            self._stream = StringIO(self._raw_post_data)
        return self._raw_post_data
    raw_post_data = property(_get_raw_post_data)
```

抛出异常的地方的逻辑是比较清晰了，就是如果没有缓存`self._raw_post_data`，同时`self._read_started`，就会抛出错误。前者显然就是缓存的结果（body的原始字符串）。后者如果看了相关代码，如果没有缓存结果时，就会从stream（HttpRequest本身就是stream）中读取结果。开始读的时候，就会设置`self._read_started`为True。看这段代码，是不会出现`self._raw_post_data`没有定义，但是`self._read_started`为True的。除非，在`self.read`里设置`self._read_started`为True后，抛异常，并且在调用方catch住了，继续处理。我看日志里并没有相关信息。    

那么，调用`request.POST`时候，干了啥勾当呢？当然，request本身是`core/handler/wsgi.py`里的WSGIRequest类型的，那部分代码这里就省略了，因为调用`request.POST`之后，通过`get_post`还是会回到`http.HttpRequest`这里的方法：`_load_post_and_files`。感觉坑就在这里了。  

```python
    def _load_post_and_files(self):
        # Populates self._post and self._files
        if self.method != 'POST':
            self._post, self._files = QueryDict('', encoding=self._encoding), MultiValueDict()
            return
        if self._read_started and not hasattr(self, '_raw_post_data'):
            self._mark_post_parse_error()
            return

        if self.META.get('CONTENT_TYPE', '').startswith('multipart'):
            if hasattr(self, '_raw_post_data'):
                # Use already read data
                data = StringIO(self._raw_post_data)
            else:
                data = self #看这里
            try:
                self._post, self._files = self.parse_file_upload(self.META, data) #看这里
            except:
                # An error occured while parsing POST data.  Since when
                # formatting the error the request handler might access
                # self.POST, set self._post and self._file to prevent
                # attempts to parse POST data again.
                # Mark that an error occured.  This allows self.__repr__ to
                # be explicit about it instead of simply representing an
                # empty POST
                self._mark_post_parse_error()
                raise
        else:
            self._post, self._files = QueryDict(self.raw_post_data, encoding=self._encoding), MultiValueDict()

```

这里，大意就是，通过`get_raw_post_data`获取body的原文后，根据Content-Type和Content-Length，解析成`self._post`。如果是这样就好了。  

但是请关注当Content-Type以"multipart"开头时，如果已经缓存了结果`_raw_post_data`，就把它转化成StringIO，交给parse_file_upload。如果没有缓存，特么的这里直接把HttpRequest对象作为StringIO交给`self.parse_file_upload`，然后设置了`_post`，然后就把`_raw_post_data`晾到一边了。

然而，在`self.parse_file_upload`之中，读取这个StringIO时，调用read方法，就会设置`self._read_started`为True。但是，`self._raw_post_data`并没有被缓存，仍然没有定义。  

# 结论

梳理一下，当请求的Content-Type为“multipart*”时，如果先调用了request.POST，然后调用request.raw_post_data，就必然会遇到这个Exception:  Exception("You cannot access raw_post_data after reading from request's data stream")



# 如何解决

当你在调用request.POST之前，就调用request.raw_post_data，这样，缓存就会被保存下来，stream就不会被再次读取，也就不会有问题了。

