title: Android上传图片，PHP接收
tags:
  - Android
  - Java
  - PHP
categories:
  - Programmer
date: 2016-04-06 21:39:13
---

上传图片到服务器是一个很常见的需求，这里给出一种简单的实现方式：
<!--more-->

# 服务器端

```php
<?php
$target_path  = "./upload/";//接收文件目录
$target_path = $target_path.basename( $_FILES['uploadedfile']['name']);
if(move_uploaded_file($_FILES['uploadedfile']['tmp_name'], $target_path)) {
   echo "The file ".basename( $_FILES['uploadedfile']['name'])." has been uploaded";
}  else {
   echo "There was an error uploading the file, please try again!".$_FILES['uploadedfile']['error'];
}
```
上面的代码保存为 `FileUploader.php`,放入服务器的`api\datas`目录，并在该目录下新建 `upload`文件夹；当然你也可以直接在PHP代码中检查是否有`upload`目录，没有可以直接使用代码新建。
这样操作之后，我们的要上传文件操作的URL就为：`http://dxjia.cn/api/datas/FileUploader.php`，你也可以按照此方法来制定你自己的访问URL。

# Android客户端
测试图片路径直接hardcode写死啦:

```java
    private static final String srcPath = "/sdcard/test.jpg";
    private static final String actionUrl = "http://dxjia.cn/api/datas/FileUpload.php";
```

绑定按钮

```
        mUpdateHandler = new UpdateHandler(this);

        testButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                doAction();
            }
        });
```

网络操作不能在主线程中进行，需要新开thread

```
    private void doAction() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                uploadFile(actionUrl);
            }
        }).start();
    }
```

主体函数，完成文件上传post，注意`form-data; name=\"uploadedfile\"`，跟服务器端的必须保持一致。

```
    /* 上传文件至Server，uploadUrl：接收文件的处理页面 */
    private void uploadFile(String uploadUrl) {
        mUpdateHandler.sendEmptyMessage(EVENT_POST_START);
        String end = "\r\n";
        String twoHyphens = "--";
        String boundary = "******";
        try {
            URL url = new URL(uploadUrl);
            HttpURLConnection httpURLConnection = (HttpURLConnection) url
                    .openConnection();
            // 允许输入输出流
            httpURLConnection.setDoInput(true);
            httpURLConnection.setDoOutput(true);
            httpURLConnection.setUseCaches(false);
            // 使用POST方法
            httpURLConnection.setRequestMethod("POST");
            httpURLConnection.setRequestProperty("Connection", "Keep-Alive");
            httpURLConnection.setRequestProperty("Charset", "UTF-8");
            httpURLConnection.setRequestProperty("Content-Type",
                    "multipart/form-data;boundary=" + boundary);

            DataOutputStream dos = new DataOutputStream(
                    httpURLConnection.getOutputStream());
            dos.writeBytes(twoHyphens + boundary + end);
            dos.writeBytes("Content-Disposition: form-data; name=\"uploadedfile\"; filename=\""
                    + srcPath.substring(srcPath.lastIndexOf("/") + 1)
                    + "\""
                    + end);
            dos.writeBytes(end);

            FileInputStream fis = new FileInputStream(srcPath);
            byte[] buffer = new byte[8192]; // 8k
            int count = 0;
            // 读取文件
            while ((count = fis.read(buffer)) != -1) {
                dos.write(buffer, 0, count);
            }
            fis.close();

            dos.writeBytes(end);
            dos.writeBytes(twoHyphens + boundary + twoHyphens + end);
            dos.flush();

            InputStream is = httpURLConnection.getInputStream();
            InputStreamReader isr = new InputStreamReader(is, "utf-8");
            BufferedReader br = new BufferedReader(isr);
            String result = br.readLine();
            Log.d("dxjia", result);

            mUpdateHandler.sendEmptyMessage(EVENT_POST_SUCCESS);

            dos.close();
            is.close();

        } catch (Exception e) {
            mUpdateHandler.sendEmptyMessage(EVENT_POST_FAILED);
            e.printStackTrace();
        }
    }
```

更新UI的handler

```
    /**
     * UI update handler
     */
    private class UpdateHandler extends Handler {
        private final Context mContext;

        public UpdateHandler(Context context) {
            mContext = context;
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case EVENT_POST_START:
                    resultTextView.setText("Started! uploading...");
                    break;
                case EVENT_POST_FAILED:
                    resultTextView.setText("upload failed!");
                    break;
                case EVENT_POST_SUCCESS:
                    resultTextView.setText("upload done!");
                    break;
            }

            super.handleMessage(msg);
        }
    }
```

# Reference
[1]. http://blog.csdn.net/sxwyf248/article/details/7012496
[2]. http://blog.csdn.net/fancylovejava/article/details/13506745
