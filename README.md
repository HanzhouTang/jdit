# Just Do It (jdit)

![Liberapay goal progress](https://img.shields.io/badge/goal%20progress-95%25-green.svg)
![Packagist](https://img.shields.io/packagist/l/doctrine/orm.svg)

**Jdit** is a research processing oriented framework based on pytorch. Only care about your ideas. 
You don't need to build a long boring code to run a deep learning project to verify your ideas.

You only need to implement you ideas and 
don't do anything with training framework, multiply-gpus, checkpoint, process visualization, performance evaluation and so on.
## Structure
There are four main module in this framework. They are `dataset`, `model`, `optimizer` and `trainer`.
Each of them are highly independent. So, you can process them easily and flexibly.
###  Dataset
First of all, for dataset, every thing is inherit from super class `Dataloaders_factory` 
from `jdit/dataset.py`, which is as following.


```python
class Dataloaders_factory(metaclass=ABCMeta):

    def __init__(self, root, batch_size=128, num_workers=-1, shuffle=True):
        """set config to `.self` """
        self.buildTransforms()
        self.buildDatasets()
        self.buildLoaders()

    def buildLoaders(self):
        """using dataset to build dataloaders"""
        
    @abstractmethod
    def buildDatasets(self):
        """rewrite this function to register 
        `self.dataset_train`,``self.dataset_valid``and ``self.dataset_test``
        """

    def buildTransforms(self, resize=32):
        """rewrite this function to register `self.train_transform_list`. 
        Default set available.
        """
        self.train_transform_list = self.valid_transform_list = [
            transforms.Resize(resize),
            transforms.ToTensor(),
            transforms.Normalize([0.5, 0.5, 0.5], [0.5, 0.5, 0.5])]

    @property
    def configure(self):
        """the info which you can get from `configure[key]` are
        "dataset_name", "batch_size", "shuffle", "root", "num_workers",
        "train_nsteps", "valid_nsteps", "test_nsteps", 
        "dataset_train", "dataset_valid","dataset_test"
        """

```
To build your dataset class, including train, valid and test. You need to do as following. 
* Define datasets. (If you don't define test dataset, it will be replaced by valid datasaet)
* Define transforms. (Default is available)

Example:
Define a datasets by using `FashionMNIST()`. 

Using default transform.

Don't define test dataset and using valid dataset  instead of test dataset. 

```python
class Fashion_mnist(Dataloaders_factory):
    def __init__(self, root=r'.\datasets\fashion_data', batch_size=128, num_workers=-1):

        super(Fashion_mnist, self).__init__(root, batch_size, num_workers)

    def buildDatasets(self):
        self.dataset_train = datasets.FashionMNIST(self.root, train=True, download=True,
                                                   transform=transforms.Compose(self.train_transform_list))
        self.dataset_valid = datasets.FashionMNIST(self.root, train=False, download=True,
                                                   transform=transforms.Compose(self.valid_transform_list))
```

###  Model
Second, you need to wrap your own network by class `Model` in `jdit/model.py`.
Let's see what's inside!

```python
class Model(object):
    def __init__(self, proto_model=None, gpu_ids_abs=(), init_method="kaiming", show_structure=False):
    """ 
        if you pass a `proto_model`, it will use class `self.define()` to init it.
        for `init_method`. you can pass a method name like `kaiming` or `xavair`       
    """
        if proto_model is not None:
            self.define(...)
            
    def __call__(self, *args, **kwargs):
        """this allows you to call model forward directly using `Model(x)`, 
        other than `Model.model(x)`  
        """
        return self.model(*args, **kwargs)

    def __getattr__(self, item):
        """this is a delegate for calling some pytorch module methods."""        
        return getattr(self.model, item)
        
    def define(self, proto_model, gpu_ids, init_method, show_structure):
    """ to define a pytorch model to Model. In other words, this is a assemble method.
        
        1. print network structure and total number of parameters.
        2. set the model to specify device, such as cpu, gpu or gpus.
        3. apply a weight init method. You can pass "kaiming" or "Xavier" for simplicity.
        Or, you can pass your own init function.
        
        self.num_params = self.print_network(proto_model, show_structure)
        self.model = self._set_device(proto_model, gpu_ids)
        init_name = self._apply_weight_init(init_method, proto_model)
    """ 
    
    def print_network(self, net, show_structure=False):
    """print total number of parameters and structure of network"""
            
    def loadModel(self, model_or_path, weights_or_path=None, gpu_ids=(), is_eval=True):
    """to assemble a model and weights from paths or passing parameters."""
    
    def loadPoint(self, model_name, epoch, logdir="log"):
    """load model and weights from a certain checkpoint. cooperate with `checkPoint()`"""
    
    def checkPoint(self, model_name, epoch, logdir="log"):
    """save model and weights for a checkpoint. cooperate with `loadPoint()`"""
    
    def countParams(self, proto_model):
    """count the total parameters of model."""
    
    @property
    def configure(self):
        """the info which you can get from `configure[key]` are
        "model_name", "init_method", "gpus", "total_params","structure"
        """
  
```
To wrap your pytorch model. You need to do as following. 
* Wrap a pytoch model in your code.
    * Using `Model(resnet18())`, to init your model.
    * Using `Model()` to get a `None` model.
  Then, using  `Model.define(resnet18())` other place to init your model.
* Load pytoch model from a file.
    * Using `Model.loadModel(model_or_path, weight_or_path)`, to load your model.
    * You must pass a model to this method whether it is path or model.
    * For `weight_or_path`, if it is not None. 
   It can be a path or weight OrderedDict and it will be applied in model.
* Do checkPoint.
    * Using `checkPoint(model_name, epoch, logdir="log")` to save your model checkpoint in `./log/checkpoint/`.
    * The Filename is `Weights_{model_name}_{epoch}.pth` and `Model_{model_name}_{epoch}.pth`
    * The `loadPoint()` is exact the opposite.
    
Example:

Load a `resnet18()` from `torchvision`.

```python
from torchvision.models.resnet import resnet18
net = Model(resnet18(), gpu_ids_abs=[], init_method="kaiming")
net.print_network()
```

###  Optimizer
Third, you need to build your own optimizer class `Optimizer` in `jdit/optimizer.py`. Let's see what's inside!
```python
class Optimizer(object):
    def __init__(self, params, lr=1e-3, lr_decay=0.92, weight_decay=2e-5, momentum=0., betas=(0.9, 0.999),
                 opt_name="Adam"):
    
    def __getattr__(self, item):
    """this is a delegate for calling some pytorch optimizer methods."""        
        return getattr(self.opt, item)
        
    def do_lr_decay(self, reset_lr_decay=None, reset_lr=None):
    """decay learning rate by `self.lr_decay`. reset `lr` and `lr_decay`
        if not None, reset `lr_decay` and `reset_lr`.
    """

    @property
    def configure(self):
     """the info which you can get from `configure[key]` are
        "opt_name", "lr_decay", other optimizer hyper-parameter, such as "weight_decay", "momentum", "betas".
        """
```
To build your optimizer method. You need to do as following. 
* Build an Optimizer by passing a series of parameters.
* Learning rate decay.
    * Using `optimizer.do_lr_deacy()` to multiply learning rate by `optimizer.lr_decay`, which you have inited before.
    * Reset learning and decay by passing the parameters to `optimizer.do_lr_deacy(reset_lr_decay=None, reset_lr=None)`

Example:

Build a adam optimizer by `Optimizer()` class.

```python

net = model()
lr = 1e-3
lr_decay = 0.94 
weight_decay = 0 
momentum = 0
betas = (0.9, 0.999)
opt_name = "RMSprop"
opt = Optimizer(net.parameters(), lr, lr_decay, weight_decay, momentum, betas, opt_name)
```


## Feature Work

