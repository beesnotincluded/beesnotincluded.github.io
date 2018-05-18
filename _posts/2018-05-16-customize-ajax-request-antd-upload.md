---
layout: post
title:  "Custom AJAX request for Ant Design Upload Component"
date:   2018-05-15 21:51:33 -0600
categories: tip
tags: antd javascript react
---
If you want to implement your own upload AJAX handler for the React [Ant Design Library Upload or Dragger components](https://ant.design/components/upload/) 
then the docs may not be much help. The missing details are below:

{% highlight javascript %}
{%- raw -%}

class UploadFile extends React.Component {
  render() {
    const draggerProps = {
      name: "file",
      multiple: false,
      showUploadList: false,
      customRequest: ({onSuccess, onError, file}) => {
        fetch('/upload', {
          method: 'POST',
          ...
          success: (resp) => {
            onSuccess()
          },
          failure: (err) => {
            onError()
          }
        )
      }
    }
    return (
        <Upload.Dragger {...draggerProps}>
          <p classname="ant-upload-drag-icon">
            <Icon type="cloud-upload"/>
             Drag & Drop a file           
          </p>
        </Upload.Dragger>
    )
  }
}
{% endraw %}
{% endhighlight %}

You just need to implement a customRequest function that accepts an object containing two callback functions: __onSuccess__, 
__onError__ and the __file__ to be uploaded. You just need to do whatever custom upload stuff you have to do in this function 
and then call either onSuccess() or onError() as appropriate when you get your response back. You don't have to pass any 
parameters to these functions.


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
