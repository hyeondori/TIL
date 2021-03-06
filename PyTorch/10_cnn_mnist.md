*[본 포스팅은 김성훈 교수님의 PyTorch Zero To All Lecture을 보며 작성한 글입니다]*

## 들어가기 전에...

신경망 아키텍쳐 중 가장 유명한 모델인 CNN에 대해 알아보자.

CNN은 이미지처리, 자연어처리에서 상당히 좋은 성능을 보여준다.

![image-20210923093045802](\assets\images\typora-user-images\image-20210923093111376.png)

CNN을 한장의 이미지로 요약하면 위와 같이 볼 수 있다. 32x32의 size를 가지는 1장의 이미지를 input-data로 넣고 5x5 size의 convolution 레이어를 통과 시키면, 우리는 28x28 size의 많은 feature maps를 `generate`할 수 있다. 여기서 2x2 size `subsampling`을 통해 convolution의 결과에 따른 information을 reduce할 수 있다. 이후에 이와 같은 convolution-subsampling을 반복하는 과정을 거치게 된다. 이러한 과정의 결과물을 `Linear model`에 input하게 되는데, 이를 우리는 `fully connected`라고 부르기도 하고, 다른 말로 `Dense Net`이라 일컫기도 한다. **결과적으로 어떠한 이미지에 따라 Output을 예측하게 된다.**

자, 갑자기 각종 어려운 용어들이 우후죽순 등장해서 어지러웠을 당신을 위해 하나하나 짚어가도록 하자.

## Convolutions

![image-20210923101412164](\assets\images\typora-user-images\image-20210923101412164.png)

convolution layer는 위의 그림과 같이 너비와 높이 그리고 깊이로 구성되는데 여기서 너비와 높이는 대개 이미지 크기보다 작게 설정을 한다. 깊이의 경우, 이미지는 RGB의 3가지 색 정보를 갖고 있기 때문에 3으로 설정한다.

**Convolutions의 핵심은 이미지로 부터 작은 부분을 보기 위함에 있다.** 그래서 우리는 `Patch`라고 부르는 이미지보다 작은 필터를 사용하는데, 우리는 이 Patch를 통해 어떠한 값을 얻을 수 있다. 물론, 이 값들은 전체 이미지에 대해 얻을 수 있지만 **한번의 과정에서는 하나의 작은 부분만**을 보게 된다.

![image-20210923132720003](\assets\images\typora-user-images\image-20210923132720003.png)

예컨대 3x3 size 이미지가 존재한다고 할 때, 2x2 size 필터를 사용한다는 예시를 들어보자. 2x2는 3x3보다 작기 때문에 작은 부분을 보게된다. 우리는 이 필터를 옆으로 움직일 수 있는데, `CNN`에서 이 과정을 `Stride`라고 표현한다. **stride: 1x1라고 표현하면, 필터를 움직이는 정도를 양옆과 위아래로 한칸씩 움직일 것이라는 말이 된다.** Stride를 진행하게 되면 필터를 옆으로 한칸씩 움직이게 되는데, 더이상 옆으로 움직일 수 없는 상태일 때 필터를 아래로 이동시킨다.

이렇게 필터를 사용하였을 때, 우리는 Feature들을 얻을 수 있다. 그러나, 각 convolution layer를 거칠 때마다 이미지 크기가 줄어듦을 알 수 있는데, 이를 방지하기 위해 우리는 `padding`이라는 기법을 활용한다.

![image-20210923132749263](\assets\images\typora-user-images\image-20210923132749263.png)

padding은 위의 그림과 같이 0으로 이루어진 값들을 이미지의 끝에 추가하여 붙이는 것을 의미한다. 이를 통해 convolution layer를 거치더라도 같은 size의 값들을 얻을 수 있다.

## Sub sampling - Pooling

![image-20210923133816684](\assets\images\typora-user-images\image-20210923133816684.png)

`Max pooling`은 상당히 간단하다. `filter size`와 `stride`에 따라 해당 필터에 해당하는 값 중 가장 큰 값을 다음 레이어로 pooling하는 과정을 말한다.

이외에도 `Average pooling` 등 `Pooling`의 종류는 다양하나 이를 진행하는 이유는 다음과 같다.

1. input size를 줄이기 위해
2. overfitting을 방지하기 위해
3. feature extraction을 더욱 용이하기 하기 위해

한번 더 요약하자면, **이미지의 feature를 뽑아내기 위해 이미지 내의 모든 정보를 활용할 필요는 없다**는 것이다. 오히려 너무 많은 정보는 연산량만 늘리고 과적합을 일으키며 적절하지 못한 feature를 얻게 되는 결론을 얻게 된다.

자, 그럼 우리는 이미지 분류를 위한 CNN 모델을 만들어보자. 준비된 데이터셋은 전통적으로 많이 활용하는 MNIST 데이터셋이다.


```python
# https://github.com/pytorch/examples/blob/master/mnist/main.py
from __future__ import print_function
import argparse
import torch
import torch.nn as nn
import torch.nn.functional as F
import torch.optim as optim
from torchvision import datasets, transforms
from torch.autograd import Variable
```

# Training settings


```python
batch_size = 64
```


```python
# MNIST Dataset
train_dataset = datasets.MNIST(root='./data/',
                               train=True,
                               transform=transforms.ToTensor(),
                               download=True)

test_dataset = datasets.MNIST(root='./data/',
                              train=False,
                              transform=transforms.ToTensor())
```

    Downloading http://yann.lecun.com/exdb/mnist/train-images-idx3-ubyte.gz


    1.7%
    
    Downloading http://yann.lecun.com/exdb/mnist/train-images-idx3-ubyte.gz to ./data/MNIST\raw\train-images-idx3-ubyte.gz


    75.2%IOPub message rate exceeded.
    The Jupyter server will temporarily stop sending output
    to the client in order to avoid crashing it.
    To change this limit, set the config variable
    `--ServerApp.iopub_msg_rate_limit`.
    
    Current values:
    ServerApp.iopub_msg_rate_limit=1000.0 (msgs/sec)
    ServerApp.rate_limit_window=3.0 (secs)
    
    100.0%


    Extracting ./data/MNIST\raw\train-images-idx3-ubyte.gz to ./data/MNIST\raw


    3.5%


​    
    Downloading http://yann.lecun.com/exdb/mnist/train-labels-idx1-ubyte.gz
    Downloading http://yann.lecun.com/exdb/mnist/train-labels-idx1-ubyte.gz to ./data/MNIST\raw\train-labels-idx1-ubyte.gz


    102.8%


    Extracting ./data/MNIST\raw\train-labels-idx1-ubyte.gz to ./data/MNIST\raw
    
    Downloading http://yann.lecun.com/exdb/mnist/t10k-images-idx3-ubyte.gz
    Downloading http://yann.lecun.com/exdb/mnist/t10k-images-idx3-ubyte.gz to ./data/MNIST\raw\t10k-images-idx3-ubyte.gz


    95.6%IOPub message rate exceeded.
    The Jupyter server will temporarily stop sending output
    to the client in order to avoid crashing it.
    To change this limit, set the config variable
    `--ServerApp.iopub_msg_rate_limit`.
    
    Current values:
    ServerApp.iopub_msg_rate_limit=1000.0 (msgs/sec)
    ServerApp.rate_limit_window=3.0 (secs)


​    

# Data Loader (Input Pipeline)


```python
train_loader = torch.utils.data.DataLoader(dataset=train_dataset,
                                           batch_size=batch_size,
                                           shuffle=True,
                                           pin_memory=True)

test_loader = torch.utils.data.DataLoader(dataset=test_dataset,
                                          batch_size=batch_size,
                                          shuffle=False,
                                          pin_memory=True)
```


```python
class Net(nn.Module):

    def __init__(self):
        super(Net, self).__init__()
        self.conv1 = nn.Conv2d(1, 10, kernel_size=5)
        self.conv2 = nn.Conv2d(10, 20, kernel_size=5)
        self.mp = nn.MaxPool2d(2)
        self.fc = nn.Linear(320, 10)

    def forward(self, x):
        in_size = x.size(0)
        x = F.relu(self.mp(self.conv1(x)))
        x = F.relu(self.mp(self.conv2(x)))
        x = x.view(in_size, -1)  # flatten the tensor
        x = self.fc(x)
        return F.log_softmax(x)


model = Net()

optimizer = optim.SGD(model.parameters(), lr=0.01, momentum=0.5)
```

여기서 우리가 주의해야할 사항은 `Linear model`에 어떤 숫자를 넣는가에 대한 것이다.
우리는 MNIST 이미지를 몇개의 레이어들을 통과시킨 후 나온 Output을 다시 Linear model을 통과시켜 10개의 Output을 얻어내야 한다.

계산 결과 320이라는 결과를 얻어서 (320, 10)으로 설정할 수도 있지만, 친절하게도 아무 숫자를 넣고 돌리면 에러 메세지와 함께 올바른 숫자를 알려준다.
PyTorch 최고!


```python
def train(epoch):
    model.train()
    for batch_idx, (data, target) in enumerate(train_loader):
        data, target = Variable(data), Variable(target)
        optimizer.zero_grad()
        output = model(data)
        loss = F.nll_loss(output, target)
        loss.backward()
        optimizer.step()
        if batch_idx % 100 == 0:
            print('Train Epoch: {} [{}/{} ({:.0f}%)]\tLoss: {:.6f}'.format(
                epoch, batch_idx * len(data), len(train_loader.dataset),
                100. * batch_idx / len(train_loader), loss.item()))
```


```python
def test():
    model.eval()
    test_loss = 0
    correct = 0
    for data, target in test_loader:
        data, target = Variable(data, volatile=True), Variable(target)
        output = model(data)
        # sum up batch loss
        test_loss += F.nll_loss(output, target, size_average=False).data
        # get the index of the max log-probability
        pred = output.data.max(1, keepdim=True)[1]
        correct += pred.eq(target.data.view_as(pred)).cpu().sum()

    test_loss /= len(test_loader.dataset)
    print('\nTest set: Average loss: {:.4f}, Accuracy: {}/{} ({:.0f}%)\n'.format(
        test_loss, correct, len(test_loader.dataset),
        100. * correct / len(test_loader.dataset)))
```


```python
for epoch in range(1, 10):
    train(epoch)
    test()
```

    C:\Users\AITHEN~1\AppData\Local\Temp/ipykernel_21268/1042560788.py:16: UserWarning: Implicit dimension choice for log_softmax has been deprecated. Change the call to include dim=X as an argument.
      return F.log_softmax(x)


    Train Epoch: 1 [0/60000 (0%)]	Loss: 2.309000
    Train Epoch: 1 [6400/60000 (11%)]	Loss: 1.517593
    Train Epoch: 1 [12800/60000 (21%)]	Loss: 0.482960
    Train Epoch: 1 [19200/60000 (32%)]	Loss: 0.480964
    Train Epoch: 1 [25600/60000 (43%)]	Loss: 0.233923
    Train Epoch: 1 [32000/60000 (53%)]	Loss: 0.456881
    Train Epoch: 1 [38400/60000 (64%)]	Loss: 0.311750
    Train Epoch: 1 [44800/60000 (75%)]	Loss: 0.285469
    Train Epoch: 1 [51200/60000 (85%)]	Loss: 0.188713
    Train Epoch: 1 [57600/60000 (96%)]	Loss: 0.197928


    C:\Users\AITHEN~1\AppData\Local\Temp/ipykernel_21268/2453309322.py:6: UserWarning: volatile was removed and now has no effect. Use `with torch.no_grad():` instead.
      data, target = Variable(data, volatile=True), Variable(target)


​    
    Test set: Average loss: 0.1824, Accuracy: 9461/10000 (95%)
    
    Train Epoch: 2 [0/60000 (0%)]	Loss: 0.213619
    Train Epoch: 2 [6400/60000 (11%)]	Loss: 0.234422
    Train Epoch: 2 [12800/60000 (21%)]	Loss: 0.204399
    Train Epoch: 2 [19200/60000 (32%)]	Loss: 0.130422
    Train Epoch: 2 [25600/60000 (43%)]	Loss: 0.220961
    Train Epoch: 2 [32000/60000 (53%)]	Loss: 0.059487
    Train Epoch: 2 [38400/60000 (64%)]	Loss: 0.208963
    Train Epoch: 2 [44800/60000 (75%)]	Loss: 0.161588
    Train Epoch: 2 [51200/60000 (85%)]	Loss: 0.155783
    Train Epoch: 2 [57600/60000 (96%)]	Loss: 0.113270
    
    Test set: Average loss: 0.1144, Accuracy: 9643/10000 (96%)
    
    Train Epoch: 3 [0/60000 (0%)]	Loss: 0.060303
    Train Epoch: 3 [6400/60000 (11%)]	Loss: 0.166026
    Train Epoch: 3 [12800/60000 (21%)]	Loss: 0.031227
    Train Epoch: 3 [19200/60000 (32%)]	Loss: 0.190414
    Train Epoch: 3 [25600/60000 (43%)]	Loss: 0.026332
    Train Epoch: 3 [32000/60000 (53%)]	Loss: 0.083587
    Train Epoch: 3 [38400/60000 (64%)]	Loss: 0.122717
    Train Epoch: 3 [44800/60000 (75%)]	Loss: 0.178541
    Train Epoch: 3 [51200/60000 (85%)]	Loss: 0.137785
    Train Epoch: 3 [57600/60000 (96%)]	Loss: 0.103704
    
    Test set: Average loss: 0.0934, Accuracy: 9709/10000 (97%)
    
    Train Epoch: 4 [0/60000 (0%)]	Loss: 0.120200
    Train Epoch: 4 [6400/60000 (11%)]	Loss: 0.050853
    Train Epoch: 4 [12800/60000 (21%)]	Loss: 0.224115
    Train Epoch: 4 [19200/60000 (32%)]	Loss: 0.031232
    Train Epoch: 4 [25600/60000 (43%)]	Loss: 0.046163
    Train Epoch: 4 [32000/60000 (53%)]	Loss: 0.029269
    Train Epoch: 4 [38400/60000 (64%)]	Loss: 0.040502
    Train Epoch: 4 [44800/60000 (75%)]	Loss: 0.039633
    Train Epoch: 4 [51200/60000 (85%)]	Loss: 0.112222
    Train Epoch: 4 [57600/60000 (96%)]	Loss: 0.046111
    
    Test set: Average loss: 0.0791, Accuracy: 9772/10000 (98%)
    
    Train Epoch: 5 [0/60000 (0%)]	Loss: 0.203232
    Train Epoch: 5 [6400/60000 (11%)]	Loss: 0.088945
    Train Epoch: 5 [12800/60000 (21%)]	Loss: 0.043348
    Train Epoch: 5 [19200/60000 (32%)]	Loss: 0.032045
    Train Epoch: 5 [25600/60000 (43%)]	Loss: 0.009201
    Train Epoch: 5 [32000/60000 (53%)]	Loss: 0.028286
    Train Epoch: 5 [38400/60000 (64%)]	Loss: 0.041674
    Train Epoch: 5 [44800/60000 (75%)]	Loss: 0.161303
    Train Epoch: 5 [51200/60000 (85%)]	Loss: 0.071224
    Train Epoch: 5 [57600/60000 (96%)]	Loss: 0.043872
    
    Test set: Average loss: 0.0669, Accuracy: 9797/10000 (98%)
    
    Train Epoch: 6 [0/60000 (0%)]	Loss: 0.047484
    Train Epoch: 6 [6400/60000 (11%)]	Loss: 0.159560
    Train Epoch: 6 [12800/60000 (21%)]	Loss: 0.019636
    Train Epoch: 6 [19200/60000 (32%)]	Loss: 0.118523
    Train Epoch: 6 [25600/60000 (43%)]	Loss: 0.211002
    Train Epoch: 6 [32000/60000 (53%)]	Loss: 0.044962
    Train Epoch: 6 [38400/60000 (64%)]	Loss: 0.098956
    Train Epoch: 6 [44800/60000 (75%)]	Loss: 0.045505
    Train Epoch: 6 [51200/60000 (85%)]	Loss: 0.051799
    Train Epoch: 6 [57600/60000 (96%)]	Loss: 0.054688
    
    Test set: Average loss: 0.0690, Accuracy: 9780/10000 (98%)
    
    Train Epoch: 7 [0/60000 (0%)]	Loss: 0.020694
    Train Epoch: 7 [6400/60000 (11%)]	Loss: 0.106460
    Train Epoch: 7 [12800/60000 (21%)]	Loss: 0.012754
    Train Epoch: 7 [19200/60000 (32%)]	Loss: 0.154484
    Train Epoch: 7 [25600/60000 (43%)]	Loss: 0.034155
    Train Epoch: 7 [32000/60000 (53%)]	Loss: 0.191154
    Train Epoch: 7 [38400/60000 (64%)]	Loss: 0.021115
    Train Epoch: 7 [44800/60000 (75%)]	Loss: 0.141251
    Train Epoch: 7 [51200/60000 (85%)]	Loss: 0.073384
    Train Epoch: 7 [57600/60000 (96%)]	Loss: 0.095912
    
    Test set: Average loss: 0.0613, Accuracy: 9805/10000 (98%)
    
    Train Epoch: 8 [0/60000 (0%)]	Loss: 0.110776
    Train Epoch: 8 [6400/60000 (11%)]	Loss: 0.035233
    Train Epoch: 8 [12800/60000 (21%)]	Loss: 0.035615
    Train Epoch: 8 [19200/60000 (32%)]	Loss: 0.023125
    Train Epoch: 8 [25600/60000 (43%)]	Loss: 0.131310
    Train Epoch: 8 [32000/60000 (53%)]	Loss: 0.148139
    Train Epoch: 8 [38400/60000 (64%)]	Loss: 0.120375
    Train Epoch: 8 [44800/60000 (75%)]	Loss: 0.170305
    Train Epoch: 8 [51200/60000 (85%)]	Loss: 0.020332
    Train Epoch: 8 [57600/60000 (96%)]	Loss: 0.075067
    
    Test set: Average loss: 0.0601, Accuracy: 9808/10000 (98%)
    
    Train Epoch: 9 [0/60000 (0%)]	Loss: 0.201269
    Train Epoch: 9 [6400/60000 (11%)]	Loss: 0.058152
    Train Epoch: 9 [12800/60000 (21%)]	Loss: 0.110900
    Train Epoch: 9 [19200/60000 (32%)]	Loss: 0.037854
    Train Epoch: 9 [25600/60000 (43%)]	Loss: 0.055617
    Train Epoch: 9 [32000/60000 (53%)]	Loss: 0.020852
    Train Epoch: 9 [38400/60000 (64%)]	Loss: 0.013024
    Train Epoch: 9 [44800/60000 (75%)]	Loss: 0.063958
    Train Epoch: 9 [51200/60000 (85%)]	Loss: 0.022507
    Train Epoch: 9 [57600/60000 (96%)]	Loss: 0.155778
    
    Test set: Average loss: 0.0546, Accuracy: 9834/10000 (98%)


​    
