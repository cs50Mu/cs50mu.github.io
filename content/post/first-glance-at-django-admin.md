+++
title = "first glance at django admin"
date = "2016-09-13"
slug = "2016/09/13/first-glance-at-django-admin"
Categories = []
+++

主要是一些定制的问题，其实django admin已经集成的非常好了，该有的都有了，一般开箱即用就行了。

如果你对Django Admin不熟悉的话，[这里](http://dokelung-blog.logdown.com/posts/220832-django-notes-6-manage-your-system-admin)有一篇很好的介绍。

### dependent select fields
其实就是子field的可选项是依赖它的父field的，这个需求在admin中没有找到配置方法，找到一个插件[`django-smart-selects`](https://github.com/digi604/django-smart-selects)

### model field options

- blank=True，实际控制的form的validation，允许表单中该字段为空

- null=True，控制的DB中字段的属性，null或者not null

### 控制某条记录的显示
需要在model的定义中加入：

```python
def __unicode__(self):
	return self.name
```
否则，记录显示出来的是Python object

### 在view中显示的model名称的自定义

这个名称默认就是显示的是在代码中定义的model的名称，可以用以下代码来自定义：

```python
class Meta:
	verbose_name = u'商品子类'
	verbose_name_plural = u'商品子类'
```

### 自定义app名称

参考[这里](http://stackoverflow.com/questions/612372/can-you-give-a-django-app-a-verbose-name-for-use-throughout-the-admin)

### 多个字段作为一个唯一键

需要在该表对应的model类的Meta类中增加`unique_together`定义：

```python
class Meta:
	unique_together = ('name', 'parent_category', 'sub_category')
	verbose_name = u'商品'
	verbose_name_plural = u'商品'
```

### 对显示界面的定制
比如，控制要显示的字段、哪几个字段是可以点击的、显示搜索框、分页等，这些都是可以配置的，不用自己来实现，非常方便。

```python
class ProductAdmin(admin.ModelAdmin):
    list_display = ('id', 'name', 'sku', 'barcode', 'price', 'description', 'create_time', 'update_time')
    list_display_links = ('id', 'name')
    exclude = ('sku',)
    search_fields = ('name',)
    list_per_page = 100
    ordering = ['id']
    empty_value_display = '-'
```

参考：

- [https://brobin.me/blog/2015/03/customizing-the-django-admin/](https://brobin.me/blog/2015/03/customizing-the-django-admin/)
- [https://www.webforefront.com/django/setupdjangomodelsindjangoadmin.html#prettyPhoto](https://www.webforefront.com/django/setupdjangomodelsindjangoadmin.html#prettyPhoto)  这篇讲的非常详尽，但其实都在官方文档里了，但有时候懒得一个一个去找了。

### 自定义方法（ModelAdmin methods）

Django的admin提供了一系列的方法，支持你通过重写这些方法来定制它的默认行为，比如`save_model`方法可以让你自定义入库的操作，加入一些自己的逻辑。它提供了非常多的方法，具体可以看文档，我目前只用到了一个`save_model`方法。

```python
class ProductAdmin(admin.ModelAdmin):
    list_display = ('id', 'name', 'sku', 'barcode', 'price', 'description', 'create_time', 'update_time')
    list_display_links = ('id', 'name')
    exclude = ('sku',)
    search_fields = ('name',)
    list_per_page = 100
    ordering = ['id']
    empty_value_display = '-'

    def save_model(self, request, obj, form, change):
        if not change:         # Add
            obj.save()         #  save in order to get the auto increment id
            generate_sku(obj)
            obj.save()         #  save generated sku
        else:                  # Update
#            generate_sku(obj) # sku should not be changed once the record is inserted, even if all the other fields have changed
            obj.save()

def generate_sku(obj):
    id = obj.id
    parent_category = obj.parent_category.code
    sub_category = obj.sub_category.code

    sku_parts = ['XLJ']
    sku_parts.extend([parent_category, sub_category, 'X'])
    today = date.today()
    sku_parts.append(today.strftime('%y%m%d'))
    seq_id = '%06d' % id
    sku_parts.append(seq_id)

    sku = re.sub(r'[^a-zA-Z0-9]', '', ''.join(sku_parts))
    obj.sku = sku
```

参考：

- [https://docs.djangoproject.com/en/dev/ref/contrib/admin/#modeladmin-methods](https://docs.djangoproject.com/en/dev/ref/contrib/admin/#modeladmin-methods)
