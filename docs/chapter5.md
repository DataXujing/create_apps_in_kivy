## 第五章 基于生成对抗网络(GAN)的一键现实转换二次元动画场景迁移项目(基于Kivymd)

------

### 1.CartoonGAN的Falsk部署与测试

为了后期安卓打包的方便，我们将CartoonGAN网络通过WebAPI的方式部署，这里我们采用Flask去部署我们的CartoonGAN,创建`flask_app.py`其代码如下：

```python
# encoding: utf-8

'''
dataxujing
2020-07-04

基于CartoonGAN的现实迁移二次元动画风格的风格迁移服务

基于Flask,Pytorch


'''

import os
from io import BytesIO
import numpy as np
import time
import cv2
from PIL import Image

import base64
import json
import flask
from flask import request, Flask

import torch
import torchvision.transforms as transforms
from torch.autograd import Variable
import torchvision.utils as vutils
from network.Transformer import Transformer

app = Flask(__name__)

# load pretrained model
model = Transformer()

model_dict = {}
# [宫崎骏, 细田守, 盗梦侦探（日本动画名）, 新海诚]
for model_name in ["Hayao","Hosoda","Paprika","Shinkai"]:
    # globals()['model_'+model_name] = Transformer().load_state_dict(torch.load(os.path.join("./static/model", model_name + '_net_G_float.pth'))).eval()
    
    globals()['model_'+model_name] = Transformer()
    globals()['model_'+model_name].load_state_dict(torch.load(os.path.join("./static/model", model_name + '_net_G_float.pth')))

# 测试接口
@app.route("/test/<name>",methods=["GET","POST"])
def test(name):
    return "Test Get: {}".format(name)


# model_name example: model_Hayao
@app.route("/predict/<model_name>", methods=["GET","POST"])
def transfor_cartoon(model_name):

    try:
        #解析图片数据
        image_b64 = base64.b64decode(str(request.form['image']))
        image_data = np.fromstring(image_b64, np.uint8)
        # image_data = cv2.imdecode(image_data, cv2.IMREAD_COLOR)
        # cv2.imwrite('/root/01.png', image_data)

        input_image = Image.open(BytesIO(image_data)).convert('RGB')

        # 开始识别
        model = globals()[model_name].eval()
        device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

        # 图像处理
        h = input_image.size[0]
        w = input_image.size[1]
        ratio = h *1.0 / w
        # 将长边缩小到450
        if ratio > 1:
            h = 450
            w = int(h*1.0/ratio)
        else:
            w = 450
            h = int(w * ratio)
        input_image = input_image.resize((h, w), Image.BICUBIC)
        input_image = np.asarray(input_image)
        # RGB -> BGR
        input_image = input_image[:, :, [2, 1, 0]]
        input_image = transforms.ToTensor()(input_image).unsqueeze(0)
        # preprocess, (-1, 1)
        input_image = -1 + 2 * input_image 

        if torch.cuda.is_available():
            model.cuda()
            input_image = Variable(input_image).cuda()
        else:
            model.float()
            input_image = Variable(input_image).float()

        # forward
        output_image = model(input_image)
        output_image = output_image[0]
        # BGR -> RGB
        output_image = output_image[[2, 1, 0], :, :]
        # deprocess, (0, 1)
        output_image = output_image.data.cpu().float() * 0.5 + 0.5
        # save
        # vutils.save_image(output_image, os.path.join(opt.output_dir, files[:-4] + '_' + opt.style + '.jpg'))

        # 去掉batch size维度
        output_image = output_image.squeeze()
        # 从[0,1]转化为[0,255]，再从CHW转为HWC，最后转为cv2
        output_image = output_image.mul_(255).add_(0.5).clamp_(0, 255).permute(1, 2, 0).type(torch.uint8).numpy()
        # RGB转BRG
        output_image_cv2 = cv2.cvtColor(output_image, cv2.COLOR_RGB2BGR)

        # opencv 转 base64
        image = cv2.imencode('.jpg', output_image_cv2)[1]
        base64_data = str(base64.b64encode(image))[2:-1]

        res = {"pred":base64_data,"code":"200"}
 
    except Exception as e:
        print(str(e))
        res = {'pred': "we loss", 'code': "404"}
    
    return json.dumps(res)
 
 
if __name__ == "__main__":
    app.run(debug=True,host="0.0.0.0",port=8080)

```

测试我们的Flask WebAPI,创建`client_test.py`


```python

'''
dataxujing
2020-07-04

请求Flask CartoonGAN服务的测试代码
用来验证Web服务的正常
'''


import requests
import base64
import json
from PIL import Image
import cv2
import numpy as np

#将图片数据转成base64格式
with open('./static/test_image/test.jpg', 'rb') as f:
    img = base64.b64encode(f.read()).decode()
image = []
image.append(img)
res = {"image":image}

#访问服务
# model: "model_Hayao"(宫崎骏),"model_Hosoda","model_Paprika","model_Shinkai"
url = "http://10.10.15.106:8080/predict/model_Hayao"
headers = {'user-agent':'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.131 Safari/537.36'}
res_back = requests.post(url,data=res,headers=headers, timeout=200)


# save cartoon image
state = json.loads(res_back.text).get('code')
if state == "404":
    print("we loss")
else:
    img_bs64 = json.loads(res_back.text).get('pred')

    imgString = base64.b64decode(img_bs64.encode("ascii"))
    nparr = np.fromstring(imgString,np.uint8)  
    image = cv2.imdecode(nparr,cv2.IMREAD_COLOR)

    cv2.imwrite("test_res.jpg",image)

```

测试结果如下图：

原图：

<div align=center>
<img src="../img/ch5/p1.jpg" width=600/> 
</div>
<br>


风格转化后的Cartoon:

<div align=center>
<img src="../img/ch5/p2.jpg" width=600/> 
</div>
<br>

### 2.kivymd定义主界面



### 3.Home界面的实现


### 4.UPLOADIMAGE功能的实现


### 5.DOWNLOAD功能的实现


### 6.项目总结