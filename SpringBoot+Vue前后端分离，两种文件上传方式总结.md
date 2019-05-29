---
title: Spring Boot + Vue 前后端分离，两种文件上传方式总结
date: 2019-05-03
tags: [Vue.js] 
published: true
feature: https://user-gold-cdn.xitu.io/2019/4/28/16a61e63ba1cae51?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文来自于 [https://juejin.im/post/5cc518186fb9a03234165172](https://juejin.im/post/5cc518186fb9a03234165172) 

在Vue.js 中，如果网络请求使用 axios ，并且使用了 ElementUI 库，那么一般来说，文件上传有两种不同的实现方案：

1. 通过 Ajax 实现文件上传
1. 通过 ElementUI 里边的 Upload 组件实现文件上传

两种方案，各有优缺点，我们分别来看。

# 准备工作

首先我们需要一点点准备工作，就是在后端提供一个文件上传接口，这是一个普通的 Spring Boot 项目，如下：
```
SimpleDateFormat sdf = new SimpleDateFormat("/yyyy/MM/dd/");
@PostMapping("/import")
public RespBean importData(MultipartFile file, HttpServletRequest req) throws IOException {
    String format = sdf.format(new Date());
    String realPath = req.getServletContext().getRealPath("/upload") + format;
    File folder = new File(realPath);
    if (!folder.exists()) {
        folder.mkdirs();
    }
    String oldName = file.getOriginalFilename();
    String newName = UUID.randomUUID().toString() + oldName.substring(oldName.lastIndexOf("."));
    file.transferTo(new File(folder,newName));
    String url = req.getScheme() + "://" + req.getServerName() + ":" + req.getServerPort() + "/upload" + format + newName;
    System.out.println(url);
    return RespBean.ok("上传成功!");
}
```

这里的文件上传比较简单，上传的文件按照日期进行归类，使用 UUID 给文件重命名。

**这里为了简化代码，我省略掉了异常捕获，上传结果直接返回成功，后端代码大伙可根据自己的实际情况自行修改。**

# Ajax 上传

在 Vue 中，通过 Ajax 实现文件上传，方案和传统 Ajax 实现文件上传基本上是一致的，唯一不同的是查找元素的方式。
```
<input type="file" ref="myfile">
<el-button @click="importData" type="success" size="mini" icon="el-icon-upload2">导入数据</el-button>
```

在这里，首先提供一个文件导入 input 组件，再来一个导入按钮，在导入按钮的事件中来完成导入的逻辑。

```
importData() {
  let myfile = this.$refs.myfile;
  let files = myfile.files;
  let file = files[0];
  var formData = new FormData();
  formData.append("file", file);
  this.uploadFileRequest("/system/basic/jl/import",formData).then(resp=>{
    if (resp) {
      console.log(resp);
    }
  })
}
```

关于这段上传核心逻辑，解释如下：

1. 首先利用 Vue 中的 $refs 查找到存放文件的元素。
1. type 为 file 的 input 元素内部有一个 files 数组，里边存放了所有选择的 file，由于文件上传时，文件可以多选，因此这里拿到的 files 对象是一个数组。
1. 从 files 对象中，获取自己要上传的文件，由于这里是单选，所以其实就是数组中的第一项。
1. 构造一个 FormData ，用来存放上传的数据,FormData 不可以像 Java 中的 StringBuffer 使用链式配置。
1. 构造好 FromData 后，就可以直接上传数据了，FormData 就是要上传的数据。
1. 文件上传注意两点，1. 请求方法为 post，2. 设置 `Content-Type` 为 `multipart/form-data` 。

这种文件上传方式，实际上就是传统的 Ajax 上传文件，和大家常见的 jQuery 中写法不同的是，这里元素查找的方式不一样（实际上元素查找也可以按照JavaScript 中原本的写法来实现），其他写法一模一样。这种方式是一个通用的方式，和使用哪一种前端框架无关。最后再和大家来看下封装的上传方法：
```
export const uploadFileRequest = (url, params) => {
  return axios({
    method: 'post',
    url: `${base}${url}`,
    data: params,
    headers: {
      'Content-Type': 'multipart/form-data'
    }
  });
}
```

经过这几步的配置后，前端就算上传完成了，可以进行文件上传了。

# 使用 Upload 组件

如果使用 Upload ，则需要引入 ElementUI，所以一般建议，如果使用了 ElementUI 做 UI 控件的话，则可以考虑使用 Upload 组件来实现文件上传，如果没有使用 ElementUI 的话，则不建议使用 Upload 组件，至于其他的 UI 控件，各自都有自己的文件上传组件，具体使用可以参考各自文档。
```
<el-upload
  style="display: inline"
  :show-file-list="false"
  :on-success="onSuccess"
  :on-error="onError"
  :before-upload="beforeUpload"
  action="/system/basic/jl/import">
  <el-button size="mini" type="success" :disabled="!enabledUploadBtn" :icon="uploadBtnIcon">{{btnText}}</el-button>
</el-upload>
```

1. show-file-list 表示是否展示上传文件列表，默认为true，这里设置为不展示。
1. before-upload 表示上传之前的回调，可以在该方法中，做一些准备工作，例如展示一个进度条给用户 。
1. on-success 和 on-error 分别表示上传成功和失败时候的回调，可以在这两个方法中，给用户一个相应的提示，如果有进度条，还需要在这两个方法中关闭进度条。
1. action 指文件上传地址。
1. 上传按钮的点击状态和图标都设置为变量 ，在文件上传过程中，修改上传按钮的点击状态为不可点击，同时修改图标为一个正在加载的图标 loading。
1. 上传的文本也设为变量，默认上传 button 的文本是 `数据导入` ，当开始上传后，将找个 button 上的文本修改为 `正在导入`。

相应的回调如下：
```
onSuccess(response, file, fileList) {
  this.enabledUploadBtn = true;
  this.uploadBtnIcon = 'el-icon-upload2';
  this.btnText = '数据导入';
},
onError(err, file, fileList) {
  this.enabledUploadBtn = true;
  this.uploadBtnIcon = 'el-icon-upload2';
  this.btnText = '数据导入';
},
beforeUpload(file) {
  this.enabledUploadBtn = false;
  this.uploadBtnIcon = 'el-icon-loading';
  this.btnText = '正在导入';
}
```

1. 在文件开始上传时，修改上传按钮为不可点击，同时修改上传按钮的图标和文本。
1. 文件上传成功或者失败时，修改上传按钮的状态为可以点击，同时恢复上传按钮的图标和文本。

上传效果图如下：

![](https://user-gold-cdn.xitu.io/2019/4/28/16a61e63ba1cae51?imageView2/0/w/1280/h/960/ignore-error/1)

# 总结

两种上传方式各有优缺点：

1. 第一种方式最大的优势是通用，一招鲜吃遍天，到哪里都能用，但是对于上传过程的监控，进度条的展示等等逻辑都需要自己来实现。
1. 第二种方式不够通用，因为它是 ElementUI 中的组件，得引入 ElementUI 才能使用，不过这种方式很明显有需多比较方便的回调，可以实现非常方便的处理常见的各种上传问题。
1. 常规的上传需求第二种方式可以满足，但是如果要对上传的方法进行定制，则还是建议使用第一种上传方案。

关注公众号牧码小子，专注于 Spring Boot+微服务，定期视频教程分享，关注后回复 Java ，领取松哥为你精心准备的 Java 干货！

![](https://user-gold-cdn.xitu.io/2019/4/28/16a61e641c0ec4fc?imageView2/0/w/1280/h/960/ignore-error/1)

