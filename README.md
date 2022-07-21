 # cdk-lambda-layer-builder

cdk-lambda-layer-builder is a collection of [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html#cpm) 
constructs to build Python Lambda Layers with minimum requirements on the user 
side, e.g. no bash or zip cli has to be available on the user's machine.

Amazon Lambda functions often require extra modules which can be packaged in an 
Amazon Lambda Layer. The Layer is then attached to the Lambda to make the packaged 
module easily usable in the function code. AWS CDK does not have a simple, production 
ready solution to create a Lambda Layer. This package solves this issue by providing 
a simple, yet powerful way to create Python Layers, where modules can come from PyPI 
or can be custom modules.

## Requirements
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) >= v2: installed and configured
* [Python](https://www.python.org/) >= 3.6
* [AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html#cpm) >= 2.X
* [Docker](https://docs.docker.com/get-docker/)

This construct works on Linux & MacOS and should work on Windows.

## Installation
You need to install `cdk-lambda-layer-builder` in the python environment you intend 
to use to build your stack. You can install the module directly from github with
```bash
$ python -m pip install git+https://github.com/aws-samples/aws-cdk-lambda-layer-builder.git
```
Or you can clone the repository first, then in the folder of this `README.md`, install 
the module with
```bash
$ pip install .
```

## Usage
Here is a full example for creating a Lambda Layer with two modules available on 
[PyPI](https://pypi.org/). The modules are [NumPy](https://pypi.org/project/numpy/) 
and [Requests](https://pypi.org/project/requests/). Simply call the CDK construct 
`BuildPyLayerAsset` and use its member variables (`BuildPyLayerAsset.asset_bucket` 
and `pypi_layer_asset.asset_key`) to create the Lambda Layer resource. The layer 
assets are built and packaged locally, within a Docker container. The standard 
[AWS packaging requirements](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) 
are used to create the asset:
```python
from aws_cdk import Stack, aws_lambda, Duration
from constructs import Construct
from cdk_lambda_layer_builder.constructs import BuildPyLayerAsset

class BuildLambdaLayerStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        '''
        '''
        super().__init__(scope, construct_id, **kwargs)

        # create the pipy layer
        pypi_layer_asset = BuildPyLayerAsset.from_pypi(self, 'PyPiLayerAsset',
            pypi_requirements=['numpy', 'requests'],
            py_runtime=aws_lambda.Runtime.PYTHON_3_8,
            asset_bucket=asset_bucket
        )
        pypi_layer = aws_lambda.LayerVersion(
            self,
            id='PyPiLayer',
            code=aws_lambda.Code.from_bucket(pypi_layer_asset.asset_bucket, pypi_layer_asset.asset_key),
            compatible_runtimes=[aws_lambda.Runtime.PYTHON_3_8],
            description ='PyPi python modules'
        )

        test_function = aws_lambda.Function(
            self,
            id='test',
            runtime=aws_lambda.Runtime.PYTHON_3_8,
            handler='main.lambda_handler',
            code=aws_lambda.Code.from_asset('lambda_code'),
            timeout=Duration.seconds(60),
            layers=[pypi_layer],
            retry_attempts=0,
        )
```
If you have some custom code, package it into a module (see `cdk_lambda_layer_builder/test/lib/lib1`
as an example) and use `BuildPyLayerAsset.from_modules` to build the Lambda Layer assets:
```python


from aws_cdk import Stack, aws_lambda, Duration
from constructs import Construct
from cdk_lambda_layer_builder.constructs import BuildPyLayerAsset

class BuildLambdaLayerStack(Stack):
    def __init__(self, scope: Construct, construct_id: str, **kwargs) -> None:
        '''
        '''
        super().__init__(scope, construct_id, **kwargs)

        # create a Lambda layer with two custom python modules
        module_layer_asset = BuildPyLayerAsset.from_modules(self, 'ModuleLayerAsset',
            local_module_dirs=['lib/lib1','lib/lib2'],
            py_runtime=aws_lambda.Runtime.PYTHON_3_8,
        )
        module_layer = aws_lambda.LayerVersion(
            self,
            id='ModuleLayer',
            code=aws_lambda.Code.from_bucket(module_layer_asset.asset_bucket, module_layer_asset.asset_key),
            compatible_runtimes=[aws_lambda.Runtime.PYTHON_3_8],
            description ='custom python modules lib1, lib2'
        )

        test_function = aws_lambda.Function(
            self,
            id='test',
            runtime=aws_lambda.Runtime.PYTHON_3_8,
            handler='main.lambda_handler',
            code=aws_lambda.Code.from_asset('lambda_code'),
            timeout=Duration.seconds(60),
            layers=[module_layer],
            retry_attempts=0,
        )
```
You can find an example of a full stack creating a Lambda function with a PyPI layer 
and a custom module layer in `cdk_lambda_layer_builder/test/app.py`.

## Test
Test procedure:
1. Clone the repo
2. Deploy the CDK test stack `cdk-lambda-layer-builder/test/app.py` by running `cdk deploy`
3. Go to `cdk-lambda-layer-builder/test/`
4. Run `pytest test_deployment.py`
5. Run `cdk destroy` to delete the resources created for the test
