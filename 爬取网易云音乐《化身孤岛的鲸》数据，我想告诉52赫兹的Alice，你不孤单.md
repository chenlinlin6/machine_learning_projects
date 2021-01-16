之前听到不才的《[化身孤岛的鲸](http://music.163.com/#/song?id=505446396)》，觉得是一首很温柔的歌，歌词很美很暖，反复听了很多次，底下也有很多很温暖，让人感动的评论，我很有兴趣想知道听歌的人的情感导向还有关注点大多是什么内容，于是我就去找相关爬取网易云音乐的方法，关于爬取的基本框架大概了解，但是对于网易云音乐的解密算法不是很了解，参考了知乎大神的代码：[如何爬网易云音乐的评论数？ - 平胸小仙女的回答 - 知乎](
https://www.zhihu.com/question/36081767/answer/140287795)，还有简书：[使用python3爬取网易云音乐的评论](https://www.jianshu.com/p/87c022b19669)，菜鸟分析：[爬取《后来的我们》](https://mp.weixin.qq.com/s?__biz=MjM5ODE1NDYyMA==&mid=2653384305&idx=1&sn=555246610925325035e7e9f77a7f4f9d&chksm=bd1cd6628a6b5f744ce3a2177303d47d59a3f9b13398c81072afa52b9f871b2a410bb6542106&mpshare=1&scene=1&srcid=0421h9BhYm9rikU3iuOWJ1HD%23rd)。
# 爬取数据部分
我们先查看页面的审查元素
![](https://upload-images.jianshu.io/upload_images/2338511-41bb0371ad1b52fa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到爬取的数据，其实爬取下来的数据是json格式的文件，可以用json.loads的方法去加载，得到的是一个层级字典的形式，然后可以去用键去索引相关的值，比如评论数据，点赞总数等等。关于如何找到这个元素，参考鸟分析：[爬取《后来的我们》](https://mp.weixin.qq.com/s?__biz=MjM5ODE1NDYyMA==&mid=2653384305&idx=1&sn=555246610925325035e7e9f77a7f4f9d&chksm=bd1cd6628a6b5f744ce3a2177303d47d59a3f9b13398c81072afa52b9f871b2a410bb6542106&mpshare=1&scene=1&srcid=0421h9BhYm9rikU3iuOWJ1HD%23rd)。
以下是实现的代码。
```python
# -*- coding: utf-8 -*- 
#! /usr/bin/python
# -*-coding:utf-8-*- 
# Author:zp

from Crypto.Cipher import AES
import base64
import requests
import json
import codecs
import time
import pickle
```


```python
# 头部信息
headers = {
    'User-Agent':'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.140 Safari/537.36',
    'Referer':'http://music.163.com/#/song?id=505446396',
    'Origin':'http://music.163.com',
    'Host':'music.163.com'
}
# 设置代理服务器
# proxies = {
#     'http:': 'http://121.232.146.184',
#     'https:': 'https://144.255.48.197'
# }

# offset的取值为:(评论页数-1)*20,total第一页为true，其余页为false
# first_param = '{rid:"", offset:"0", total:"true", limit:"20", csrf_token:""}' # 第一个参数
second_param = "010001"  # 第二个参数
# 第三个参数
third_param = "00e0b509f6259df8642dbc35662901477df22677ec152b5ff68ace615bb7b725152b3ab17a876aea8a5aa76d2e417629ec4ee341f56135fccf695280104e0312ecbda92557c93870114af6c9d05c4f7f0c3685b7a46bee255932575cce10b424d813cfe4875d3e82047b97ddef52741d546b8e289dc6935b3ece0462db0a22b8e7"
# 第四个参数
forth_param = "0CoJUm6Qyw8W8jud"


# 获取参数
def get_params(page):  # page为传入页数
    iv = "0102030405060708"
    first_key = forth_param
    second_key = 16 * 'F'
    if (page == 1):  # 如果为第一页
        first_param = '{rid:"", offset:"0", total:"true", limit:"20", csrf_token:""}'
        h_encText = AES_encrypt(first_param, first_key, iv)#加密方式是第一个参数和最后一个参数通过固定iv加密
    else:
        offset = str((page - 1) * 20)
        first_param = '{rid:"", offset:"%s", total:"%s", limit:"20", csrf_token:""}' % (offset, 'false')
        h_encText = AES_encrypt(first_param, first_key, iv)
    h_encText = AES_encrypt(h_encText, second_key, iv)#第二次加密是上次加密返回值与一个16位的任意字符串加密
    return h_encText


# 获取 encSecKey
def get_encSecKey():
    encSecKey = "257348aecb5e556c066de214e531faadd1c55d814f9be95fd06d6bff9f4c7a41f831f6394d5a3fd2e3881736d94a02ca919d952872e7d0a50ebfa1769a7a62d512f5f1ca21aec60bc3819a9c3ffca5eca9a0dba6d6f7249b06f5965ecfff3695b54e1c28f3f624750ed39e7de08fc8493242e26dbc4484a01c76f739e135637c"
    return encSecKey


# 解密过程
def AES_encrypt(text, key, iv):
    pad = 16 - len(text) % 16
    #print(type(text))
    text = text + pad * chr(pad)
    encryptor = AES.new(key, AES.MODE_CBC, iv)
    encrypt_text = encryptor.encrypt(text)
    encrypt_text = base64.b64encode(encrypt_text).decode()
    return encrypt_text
```


```python
# 获得评论json数据
def get_json(url, params, encSecKey):
    data = {
        "params": params,
        "encSecKey": encSecKey
    }
    response = requests.post(url, headers=headers, data=data)
    return response.content



# 抓取某一首歌的全部评论
def get_all_comments(url):
    all_comments_list = []  # 存放所有评论
    all_comments_list.append(u"用户ID 用户昵称 用户头像地址 评论时间 点赞总数 评论内容\n")  # 头部信息
    params = get_params(1)
    encSecKey = get_encSecKey()
    json_text = get_json(url, params, encSecKey).decode('utf-8')
    json_dict = json.loads(json_text)
    print 'json_dict', json_dict
    # comments_num = int(json_dict['total'])
    comments_num = 5402
    if (comments_num % 20 == 0):
        page = int(comments_num / 20)
    else:
        page = int(comments_num / 20) + 1
    # page=271
    print("共有%d页评论!" % page)
    print(type(page))
    File=codecs.open(filename, 'a', encoding='utf-8')
    for i in range(page):  # 逐页抓取
        params = get_params(i + 1)
        encSecKey = get_encSecKey()
        json_text = get_json(url, params, encSecKey).decode('utf-8')
        json_dict = json.loads(json_text)
        if i == 0:
            print("共有%d条评论!" % comments_num)  # 全部评论总数
        for item in json_dict['comments']:
            comment = item['content']  # 评论内容
            likedCount = item['likedCount']  # 点赞总数
            comment_time = item['time']  # 评论时间(时间戳)
            userID = item['user']['userId']  # 评论者id
            nickname = item['user']['nickname']  # 昵称
            avatarUrl = item['user']['avatarUrl']  # 头像地址
            comment_info = comment + u"\n"
            File.write(comment_info)
            File.flush()
            all_comments_list.append(comment_info)
        print("第%d页抓取完毕!" % (i + 1))
    File.close()
    print "所有页抓取完"
    return all_comments_list

# 抓取热门评论，返回热评列表
def get_hot_comments(url):
    file_name='hotcomments_dict.pkl'
    dict_file=codecs.open(file_name, 'r')
    hot_comments_list = []
    hot_comments_list.append(u"用户ID" + "," + u"用户昵称" + "," + u"评论时间" + "," + u"点赞总数" + "," + u"评论内容" + ","  + u"\r\n"
                            )
    params = get_params(1)  # 第一页
    encSecKey = get_encSecKey()
    json_text = get_json(url,params,encSecKey).decode('utf-8')
    json_dict = json.loads(json_text)
    hot_comments = json_dict['hotComments']
    #将hot_comments存储到pickle文件，下次我们可以调用
    pickle.dump(hot_comments, dict_file)
    # 热门评论
    #print(hot_comments)
    dict_file.close()
    print("共有%d条热门评论!" % len(hot_comments))
    for item in hot_comments:
        comment = item['content']  # 评论内容
        likedCount = item['likedCount']  # 点赞总数
        #comment_time = item['time'] / 1000 # 评论时间(时间戳)除1000转化为10位时间戳
        #print(comment_time)
        # 转换成localtime
        time_local = time.localtime(item['time'] / 1000)
        comment_time = time.strftime("%Y-%m-%d", time_local)
        userId = item['user']['userId']  # 评论者id
        nickname = item['user']['nickname']  # 昵称
        #print(type(comment))
        comment_info = str(userId) + ","  +nickname+"," +str(comment_time)+"," +str(likedCount)   +u"\t\t" +   comment + u"\r\n"
        hot_comments_list.append(comment_info)
    
    return hot_comments_list

# def save_to_file(info, File):
#     File.write(info)
#     File.flush()
    # print("写入文件成功!")


if __name__ == "__main__":
    start_time = time.time()
    url = "http://music.163.com/weapi/v1/resource/comments/R_SO_4_505446396?csrf_token"
#     filename = "化身孤岛的蓝鲸_.txt"
#     all_comments_list = get_all_comments(url)
    # save_to_file(all_comments_list, filename)
    hot_comments_list = get_hot_comments(url)
    #print(hot_comments)
    #save_to_file(hot_comments,filename)
    end_time = time.time()  # 结束时间
    print("程序耗时%f秒." % (end_time - start_time))
```
 



```python
from gensim.models.word2vec import Word2Vec
import jieba
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from wordcloud import WordCloud
%matplotlib inline
```


```python
rf=open('hotcomments_dict.pkl','r')
dict_data=pickle.load(rf)
df=pd.DataFrame(dict_data)
len(df)
```
    15
```python
df.sort_values(by='likedCount',ascending=False).iloc[0]
```




    beReplied                                                     []
    commentId                                              575831532
    content        你问抑郁症的人生活这么美好为什么会想要自杀呢，这就像你问一个哮喘的人周围全是空气你怎么回喘不到气呢
    liked                                                      False
    likedCount                                                 19871
    pendantData                                                 None
    time                                               1507132400441
    user           {u'remarkName': None, u'expertTags': None, u'a...
    Name: 0, dtype: object

最后爬取的数据
![爬取的评论文本](https://upload-images.jianshu.io/upload_images/2338511-cb1068ca2c6e8eb9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 词云显示 

我们用`wordcloud`模块来看看哪些词出现得最多。
```python
# -*- coding: utf-8 -*- 
#用utf-8的编码方式来打开爬取的评论文件
import codecs
fi=codecs.open('化身孤岛的蓝鲸_.txt', 'r', 'utf-8')
st=[u'%s'%line.strip() for line in fi]
```


```python
#打印出第一条评论看看
print st[0]
```

    这个世界上还有很多你没有吃过的美食，没见过的风景，希望你和那个叫安生的女孩一样，在漫长而枯燥的人生中执着地守望下去。——来自累的半死的初三中考党[鬼脸]



```python
print len(st)
```

    6010



```python
#使用wordcloud对象来对文本生成文字云
wordcloud=WordCloud(font_path='/Users/chenlinlin/Downloads/simhei.ttf',max_words=200).generate(''.join(st))
```


```python
#文字云可视化，imshow可以对numpy矩阵转化成可视化的图片
plt.figure()
plt.imshow(wordcloud)
```




    <matplotlib.image.AxesImage at 0x1157a1ed0>

![output_12_1.png](https://upload-images.jianshu.io/upload_images/2338511-cfbef5926c0a83ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 从上面看来，评论的人都是比较感性的，“爱心”，“可爱”，"憨笑"，“大哭”这类词从评论来看都是关于表情的词，说明很多人的评论里面都有发表情。其中大多数人用的表情都是比较正向的，“加油”，“爱心”，“你好”

# word2vec分析

`word2vec`可以用来分析一个词相关的上下文周围的词，可以计算词与词之间的相关度，是一种词义的关联度分析工具。比如说在娱乐相关的文本当中，对于一个训练好的模型，如果我输入“黄晓明”这个词，那么应该会返回跟“黄晓明”这个词最相关的一些词，比如“Angelababy”，“杨颖”，“玄奘”（他演过玄奘），“帅”，“中餐厅”等关键词，且会按照相似度或者叫相关性高低排好序。
```python
#我们用jieba分词返回每条评论分词后的列表
total_list=[jieba.lcut(line,cut_all=True) for line in st]
```

    Building prefix dict from the default dictionary ...
    Dumping model to file cache /var/folders/d3/8k7fdhls6_97llsvsb_fdg4r0000gn/T/jieba.cache
    Loading model cost 3.434 seconds.
    Prefix dict has been built succesfully.



```python
#利用分词后的列表作为训练数据，传入word2vec模型进行训练
model = Word2Vec(total_list, size=100, window=5, min_count=5, workers=4)
```


```python
#保存训练好的模型
fname='wangyiyun'
model.save(fname)
load_model=Word2Vec.load(fname)
```


```python
#我们查看一下一些词的相近词是哪些
def check_sim(noun):
    for word, score in load_model.wv.most_similar(positive=noun.decode('utf-8')):
        print word, score
```


```python
check_sim('同伴')
```

    世上 0.998196363449
    家人 0.998050451279
    海洋 0.998008489609
    尸体 0.998001933098
    尽心尽力 0.997954905033
    尽心 0.997846841812
    系 0.99779266119
    十八 0.997725129128
    所 0.997719228268
    系统 0.997692108154



```python
check_sim('52')
```

    频率 0.999604403973
    赫兹 0.999144017696
    鲸鱼 0.998784661293
    15 0.998501062393
    普通 0.998339772224
    发出 0.998339653015
    化身 0.998124480247
    只 0.998099803925
    海洋 0.998004436493
    尸体 0.997999727726



```python
check_sim('改编')
```

    名字 0.999368548393
    音乐 0.999324321747
    一种 0.999313235283
    到 0.999272465706
    出来 0.999268651009
    再也 0.9992659688
    点 0.999263226986
    不可 0.999259114265
    没人 0.999245226383
    最后 0.999242424965


我们可以看到，每个词都能返回对应的词义相近的一些词，说明这个向量训练是比较成功的

# 情感分析 

我这里预先找了些有标注好正向和负向的文本，我们用这两个表格里面的文本分词，然后利用分的词进行构建词向量，再将词向量和对应的标签用分类模型进行训练，然后训练好的分类模型可以用来预测我们实际评论的情感（正向或者是负向）。


```python
ls
```

    hotcomments_dict.pkl         x_train.npy
    [31mneg.xls[m[m*                     y_train.npy
    [31mpos.xls[m[m*                     化身孤岛的蓝鲸.ipynb
    wangyiyun                    化身孤岛的蓝鲸_.txt



```python
neg_string=pd.read_excel('neg.xls',header=None, names=['text'],index=None)
```


```python
pos_string=pd.read_excel('pos.xls',header=None, names=['text'],index=None)
```


```python
neg_string.head()
```
![输出结果](https://upload-images.jianshu.io/upload_images/2338511-82fce362498b7a2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```python
neg_list=neg_string['text'].apply(lambda x:jieba.lcut(x))
pos_list=pos_string['text'].apply(lambda x:jieba.lcut(x))
```


```python
x_train=np.concatenate((neg_list, pos_list), axis=0)
y_train=np.concatenate((np.zeros(len(neg_list)),np.ones(len(pos_list))),axis=0)
```


```python
print y_train[:5]
```

    [ 0.  0.  0.  0.  0.]



```python
np.save('x_train.npy',x_train,allow_pickle=True)
np.save('y_train.npy',y_train,allow_pickle=True)
```


```python
X_train_load=np.load('x_train.npy')
Y_train_load=np.load('y_train.npy')
```


```python
#我们得到了划分好词的词列表
print X_train_load[0]
```

    [u'\u505a', u'\u4e3a', u'\u4e00\u672c', u'\u58f0\u540d\u5728\u5916', u'\u7684', u'\u6d41\u884c', u'\u4e66', u'\uff0c', u'\u8bf4', u'\u7684', u'\u8fd8\u662f', u'\u5e7f\u5dde', u'\u7684', u'\u5916\u4f01', u'\uff0c', u'\u6309', u'\u9053\u7406', u'\u5e94\u8be5', u'\u548c', u'\u6211', u'\u7684', u'\u751f\u5b58\u73af\u5883', u'\u5dee\u4e0d\u591a', u'\u554a', u'\u3002', u'\u4f46\u662f', u'\u4e00\u770b\u4e4b\u4e0b', u'\uff0c', u'\u624d', u'\u53d1\u73b0', u'\u76f8\u53bb\u751a\u8fdc', u'\u3002', u'\u8fd9', u'\u4e5f', u'\u5c31\u7b97', u'\u4e86', u'\uff0c', u'\u8fd8', u'\u53d1\u73b0', u'\u5176\u4e2d', u'\u7684', u'\u5f88\u591a', u'\u89c4\u5219', u'\u6709', u'\u5f88', u'\u5f3a', u'\u7684', u'\u4f01\u4e1a', u'\u4e2a\u6027', u'\uff0c', u'\u4e5f', u'\u5c31', u'\u8bf4', u'\uff0c', u'\u53ea\u662f', u'\u4e2a\u4f8b', u'\uff0c', u'\u800c', u'\u4e0d\u662f', u'\u884c\u4f8b', u'\u3002', u'\u7ed9', u'\u6211\u4eec', u'\u8fd9\u4e9b', u'\u8001\u6cb9\u6761', u'\u770b\u770b', u'\u4e5f', u'\u5c31\u7b97', u'\u4e86', u'\uff0c', u'\u5982\u679c', u'\u7ed9', u'\u90a3\u4e9b', u'\u5bf9', u'\u5916\u4f01', u'\u5411\u5f80', u'\uff0c', u'\u6216\u8005', u'\u60f3', u'\u4e86\u89e3', u'\u7684', u'freshman', u'\u6765\u770b', u'\uff0c', u'\u5b9e\u5728', u'\u662f', u'\u5bb9\u6613', u'\u8bef\u5bfc', u'\u4ed6\u4eec', u'\u3002']



```python
#将分好词的文本通过空格连接起来，后面可以被tfidf方便分词，原因是tdidf不适用于中文，要给它创造空格，它才能识别分割点
corpus=[]
for i in X_train_load:
    corpus.append(' '.join(i))
```


```python
#建立矢量化对象，然后对语料库进行矢量的转化
from sklearn.feature_extraction.text import TfidfVectorizer
vectorizer=TfidfVectorizer(decode_error='ignore')
tfidf=vectorizer.fit_transform(corpus)
```


```python
key1=vectorizer.vocabulary_.keys()[2]
print key1, vectorizer.vocabulary_[key1]
```

    游乐园 31872



```python
#idf_代表着所有词的idf_值
len(vectorizer.idf_)
```




    47593




```python
vectorizer.vocabulary_这个是个字典，键是某个词，值是对应的idf_值
len(vectorizer.vocabulary_)
```




    47593




```python
words = vectorizer.get_feature_names()
key1 in words
```




    True




```python
print len(X_train_load)
```

    21105



```python
shape(tfidf)
```

通过观察返回的tfidf矩阵形状，我们大概可以知道，每一列都是一个词，每个词都是一个维度，每一行则是一条数据或者说一个文本，这是一个很稀疏的矩阵，因为一条文本很多词都没有出现


```python
for i in range(2):
    print '-'*6, '%d'%i, '-'*6
    for j in range(len(words)):
        if tfidf[i,j]>1e-5:
            print words[j].encode('utf-8'), tfidf[i,j]
```

    ------ 0 ------
    freshman 0.222534970841
    一本 0.0912320991251
    一看之下 0.217346678202
    不是 0.0784382852075
    个例 0.196991225492
    个性 0.145477100458
    了解 0.11372500084
    他们 0.0990841104203
    企业 0.137805284055
    但是 0.0785093527687
    其中 0.119239246281
    发现 0.182341339954
    只是 0.0992571949839
    向往 0.16958641363
    声名在外 0.222534970841
    外企 0.344204957845
    如果 0.0889862457552
    实在 0.0997843164797
    容易 0.108482824239
    就算 0.275915499289
    差不多 0.128735168491
    广州 0.162897541062
    应该 0.0935818042797
    很多 0.0834615934415
    我们 0.0791345444099
    或者 0.122149404381
    来看 0.129466718421
    流行 0.15501046377
    生存环境 0.213107534451
    相去甚远 0.201230369243
    看看 0.110600052122
    老油条 0.217346678202
    行例 0.222534970841
    规则 0.178136352711
    误导 0.16076293117
    还是 0.0701086316652
    这些 0.111810050365
    道理 0.130664923146
    那些 0.115171696807
    ------ 1 ------
    上班 0.249034417754
    作者 0.137569128674
    倾向 0.290285490534
    只有 0.146009953111
    可以 0.0989275316498
    太太 0.265137899367
    嫌疑 0.248188388047
    实用 0.173847399515
    很多 0.122988673898
    才能 0.178875418971
    抄袭 0.280428794382
    方法 0.185567177728
    明显 0.182608354795
    没买 0.272783346874
    没意思 0.238752095479
    生活 0.14971882704
    省省 0.327926652622
    老公 0.227042923197
    自恋 0.293266714555
    还有 0.128111999292
    那样 0.180807811169


## 训练分类模型 

接下来我们尝试用multibayes来训练这个分类模型


```python
from sklearn.naive_bayes import MultinomialNB
MB=MultinomialNB()
MB.fit(tfidf,Y_train_load)
```




    MultinomialNB(alpha=1.0, class_prior=None, fit_prior=True)




```python
#我们试一下预测前5个文本看看
print MB.predict(tfidf[:5])
```

    [ 0.  0.  0.  0.  1.]


# 处理原来的文本 


```python
df.head(2)
```
![输出结果](https://upload-images.jianshu.io/upload_images/2338511-2453549475df2387.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来我们还是对原本爬取的文本进行分词，然后连接，通过vectorizer转换成tfidf矩阵，然后给训练好的分类器预测


```python
process_st=[' '.join(jieba.cut(i,cut_all=True)) for i in st]
```


```python
print process_st[:5]
```

    [u'\u8fd9\u4e2a \u4e16\u754c \u4e0a \u8fd8\u6709 \u5f88\u591a \u4f60 \u6ca1\u6709 \u5403 \u8fc7 \u7684 \u7f8e\u98df   \u6ca1\u89c1 \u8fc7 \u7684 \u98ce\u666f   \u5e0c\u671b \u4f60 \u548c \u90a3\u4e2a \u53eb\u5b89 \u5b89\u751f \u7684 \u5973\u5b69 \u4e00\u6837   \u5728 \u6f2b\u957f \u800c \u67af\u71e5 \u7684 \u4eba\u751f \u4e2d \u6267\u7740 \u7740\u5730 \u5b88\u671b \u4e0b\u53bb     \u6765\u81ea \u7d2f \u7684 \u534a\u6b7b \u7684 \u521d\u4e09 \u4e09\u4e2d \u4e2d\u8003 \u515a   \u9b3c\u8138  ', u'\u6e29\u67d4 \u771f\u7684 \u662f \u4e00\u526f \u826f\u836f   \u5316 \u4e16\u95f4 \u75c5\u75db \u4e3a \u548c\u5e73   \u6da6 \u5e72\u6db8 \u571f\u5730 \u4e3a \u826f\u7530   \u5f00 \u4e07\u53e4 \u4e3a \u60c5\u75f4', u'\u79c1\u5954 \u5417 \u5c0f\u59d0 \u5c0f\u59d0\u59d0 \u59d0\u59d0', u'\u5b89\u751f   \u5b89\u5b89 \u5b89\u5b89\u7a33\u7a33 \u5b89\u7a33 \u7a33\u7a33 \u7684 \u5ea6\u8fc7 \u4e00\u751f \u4e5f \u5f88 \u597d   \u4f60 \u4e00\u5b9a \u5b9a\u8981 \u8d70\u51fa \u51fa\u6765   \u867d\u7136 \u6211 \u8bf4 \u8fd9 \u53e5 \u8bdd \u663e\u5f97 \u90a3\u4e48 \u65e0\u529b   \u56e0\u4e3a \u6211 \u548c \u4f60 \u4e00\u6837   \u5e38\u5e38 \u9677\u5165 \u9ed1\u6697 \u6697\u91cc   \u903c\u8d70 \u4e86 \u4e00\u4e2a \u53c8 \u4e00\u4e2a \u559c\u6b22 \u6211 \u7684 \u4eba   \u4f46\u662f   \u73b0\u5728 \u6b63\u5728 \u8d8a\u6765 \u8d8a\u6765\u8d8a \u8d8a\u597d   \u4f60 \u4e5f \u8981 \u52a0\u6cb9   \u5927\u5bb6 \u90fd \u5f88 \u7406\u89e3 \u4f60   \u8981 \u65e9\u65e5 \u65e9\u65e5\u5eb7\u590d \u5eb7\u590d', u'\u6b4c \u4e0e \u66f2 \u6709\u79cd 90 \u5e74\u5e95 \u6e2f\u53f0 \u7279\u6709 \u7684 \u4e50\u611f   \u53ef\u7231  ']


通过数据查看，是处理成功的，接下来应该是对该列转化成tfidf矩阵


```python
pre_tfidf=vectorizer.transform(process_st)
```


```python
pre_tfidf.shape
```




    (6010, 47593)




```python
predict_result=MB.predict(pre_tfidf)
print predict_result[:100]
```

    [ 1.  1.  1.  1.  1.  0.  0.  0.  1.  0.  1.  0.  1.  1.  1.  1.  1.  1.
      1.  1.  1.  1.  1.  1.  1.  0.  1.  1.  1.  1.  1.  1.  1.  1.  1.  0.
      1.  1.  1.  1.  1.  0.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.
      0.  1.  1.  0.  1.  1.  1.  1.  1.  1.  0.  1.  1.  1.  0.  0.  1.  1.
      1.  0.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.  1.
      0.  0.  1.  1.  1.  1.  1.  1.  0.  0.]







```python
final_df=pd.DataFrame({'content':st,'label':predict_result.tolist()})
```


```python
final_df.head(2)
```
![](https://upload-images.jianshu.io/upload_images/2338511-8b49c65ce5b7f507.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





```python
import seaborn as sns
# final_df['label'].plot(kind='pie')

final_df['label'].value_counts().plot(kind='pie',figsize=(10,10))
```




    <matplotlib.axes._subplots.AxesSubplot at 0x1246312d0>




![1代表积极情感，0代表消极情感](https://upload-images.jianshu.io/upload_images/2338511-914c73bfc4f57104.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



通过简单的情感分析，可以知道绝大多数人都是持正向情感的，但也有相当一部分人持消极情感


```python
print final_df[final_df['label']==0].head(10)
```

                                                  content  label
    5                                        怎么这么温柔这么好听啊…    0.0
    6   嘿，我又想你了。来听你喜欢的歌了，我们什么联系方式都没有了。关于我，你遗憾吗？多想再见你一次...    0.0
    7   谈恋爱好累，感觉还是自己一个人好，什么都不用顾照顾好自己就可以了，我也不知道过度幼稚是一个什...    0.0
    9                            过度幼稚只会让对方觉得心累，也有可能他不喜欢你了    0.0
    11  我的男朋友在家里是一个独生子，他一直说我像个小孩子，什么时候才能长大。我感觉 不是在喜欢的人...    0.0
    25                               你还欠我一句对不起、可我不会再说没关系了    0.0
    35                                    无损怎么评论才5k上面的6w…    0.0
    41                                   致正在追梦的你，又没人能懂的你。    0.0
    54  心理咨询师说我有轻度焦虑中度抑郁，我笑着问他别人轻度抑郁都会想要轻生，我都中度了怎么只是情绪...    0.0
    57                           早晨四点二十二分 失眠 你好 新的一天.[心碎]    0.0



```python
print final_df[final_df['label']==1].head(10)
```

                                                  content  label
    0   这个世界上还有很多你没有吃过的美食，没见过的风景，希望你和那个叫安生的女孩一样，在漫长而枯燥...    1.0
    1                  温柔真的是一副良药，化世间病痛为和平，润干涸土地为良田，开万古为情痴    1.0
    2                                              私奔吗小姐姐    1.0
    3   安生，安安稳稳的度过一生也很好，你一定要走出来，虽然我说这句话显得那么无力，因为我和你一样，...    1.0
    4                                歌与曲有种90年底港台特有的乐感[可爱]    1.0
    8                                    安生    要平平安安过完一生哦    1.0
    10                                     多少故事，你已不在，独我留恋    1.0
    12                                             好多人的心声    1.0
    13                             好像有粤语版的，，，好熟悉   就是想不起来    1.0
    14                                       小女子不才，让公子就等了    1.0



> 可以看到，实际上由于用于训练的数据跟预测的数据歌词还是不太匹配的，也就是歌词当中的很多词可能并没有在模型当中出现，所以生成的tfidf矩阵可能是相当稀疏的，所以预测结果还是有比较大的差异，但无论如何，本次也算是对情感分析的一次简单且完整的探索。如果以后能够
