Managing Dependencies in MLflow Models
======================================

`MLflow Model <../models.html>`_ is a standard format that packages a machine learning model with its dependencies and other metadata.
Building a model with its dependencies allows for reproducibility and portability across a variety of platforms and tools.

When you create an MLflow model using the `MLflow Tracking APIs <../tracking.html>`_, for instance, :py:func:`mlflow.pytorch.log_model`,
MLflow automatically infers the required dependencies for the model flavor you're using and records them as a part of Model metadata. Then, when you
serve the model for prediction, MLflow automatically installs the dependencies to the environment. Therefore, you normally won't need to
worry about managing dependencies in MLflow Model.

However, in some cases, you may need to add or modify some dependencies. This page provides a high-level description of how MLflow manages
dependencies and guidance for how to customize dependencies for your use case.

.. tip::

    One tip for improving MLflow's dependency inference accuracy is to add an ``input_example`` when saving your model. This enables MLflow to 
    perform a model prediction before saving the model, thereby capturing the dependencies used during the prediction.
    Please refer to :ref:`Model Input Example <input-example>` for additional, detailed usage of this parameter.

.. contents:: Table of Contents
  :local:
  :depth: 1

.. _how-mlflow-records-dependencies:

How MLflow Records Model Dependencies
-------------------------------------

An MLflow Model is saved within a specified directory with the following structure:

::

    my_model/
    ├── MLmodel
    ├── model.pkl
    ├── conda.yaml
    ├── python_env.yaml
    └── requirements.txt

Model dependencies are defined by the following files (For other files, please refer to the guidance provided in the section discussing :ref:`Storage Format <model-storage-format>`):

* ``python_env.yaml`` - This file contains the information required to restore the model environment using virtualenv (1) python version (2) build tools like pip, setuptools, and wheel (3) pip requirements of the model (a reference to requirements.txt)
* ``requirements.txt`` - Defines the set of pip dependencies required to run the model.
* ``conda.yaml`` - Defines the conda environment required to run the model. This is used when you specify ``conda`` as the environment manager for restoring the model environment.

Please note that **it is not recommended to edit these files manually** to add or remove dependencies.
They are automatically generated by MLflow and any change you make manually will be overwritten when you save the model again.
Instead, you should use one of the recommended methods described in the following sections.

Example
~~~~~~~

The following shows an example of environment files generated by MLflow when logging a model with ``mlflow.sklearn.log_model``:

* ``python_env.yaml``

  .. code-block:: yaml

      python: 3.9.8
      build_dependencies:
      - pip==23.3.2
      - setuptools==69.0.3
      - wheel==0.42.0
      dependencies:
      - -r requirements.txt

* ``requirements.txt``

  .. code-block:: text

      mlflow==2.9.2
      scikit-learn==1.3.2
      cloudpickle==3.0.0

* ``conda.yaml``

  .. code-block:: yaml

      name: mlflow-env
      channels:
        - conda-forge
      dependencies:
      - python=3.9.8
      - pip
      - pip:
        - mlflow==2.9.2
        - scikit-learn==1.3.2
        - cloudpickle==3.0.0


Adding Extra Dependencies to an MLflow Model
--------------------------------------------
MLflow infers dependencies required for the model flavor library, but your model may depend on other libraries e.g. data
preprocessing. In this case, you can add extra dependencies to the model by specifying the **extra_pip_requirements** param
when logging the model. For example,

.. code-block:: python

    import mlflow


    class CustomModel(mlflow.pyfunc.PythonModel):
        def predict(self, context, model_input):
            # your model depends on pandas
            import pandas as pd

            ...
            return prediction


    # Log the model
    with mlflow.start_run() as run:
        mlflow.pyfunc.log_model(
            python_model=CustomModel(),
            artifact_path="model",
            extra_pip_requirements=["pandas==2.0.3"],
            input_example=input_data,
        )

The extra dependencies will be added to ``requirements.txt`` as follows (and similarly to ``conda.yaml``):

.. code-block:: yaml

    mlflow==2.9.2
    cloudpickle==3.0.0
    pandas==2.0.3  # added


In this case, MLflow will install Pandas 2.0.3 in addition to the inferred dependencies when serving the model for prediction.

.. note::

    Once you log the model with dependencies, it is advisable to test it in a sandbox environment to avoid any dependency
    issues when deploying the model to production. Since MLflow 2.10.0, you can use the :py:func:`mlflow.models.predict()` API to quickly test
    your model in a virtual environment. Please refer to :ref:`Validating Environment for Prediction <validating-environment-for-prediction>` for more details.

Defining All Dependencies by Yourself
-------------------------------------

Alternatively, you can also define all dependencies from scratch rather than adding extra ones. To do so,
specify **pip_requirements** when logging the model. For example,

.. code-block:: python

    import mlflow

    # Log the model
    with mlflow.start_run() as run:
        mlflow.sklearn.log_model(
            sk_model=model,
            artifact_path="model",
            pip_requirements=[
                "mlflow-skinny==2.9.2",
                "cloudpickle==2.5.8",
                "scikit-learn==1.3.1",
            ],
        )

The manually defined dependencies will override the default ones MLflow detects from the model flavor library:

.. code-block:: yaml

    mlflow-skinny==2.9.2
    cloudpickle==2.5.8
    scikit-learn==1.3.1

.. warning::

    Please be careful when declaring dependencies that are different from those used during training, as it can be dangerous
    and prone to unexpected behavior. The safest way to ensure consistency is to rely on the default dependencies inferred by MLflow.

.. note::

    Once you log the model with dependencies, it is advisable to test it in a sandbox environment to avoid any dependency
    issues when deploying the model to production. Since MLflow 2.10.0, you can use the :py:func:`mlflow.models.predict()` API to quickly
    test your model in a virtual environment. Please refer to :ref:`Validating Environment for Prediction <validating-environment-for-prediction>` for more details.


Saving Extra Code with an MLflow Model
--------------------------------------
MLflow also supports saving your custom Python code as dependencies to the model. This is particularly useful
when you want to deploy your custom modules that are required for prediction with the model.
To do so, specify **code_path** when logging the model. For example, if you have the following file structure in your project:

::

    my_project/
    ├── utils.py
    └── train.py

.. code-block:: python
    :caption: train.py

    import mlflow


    class MyModel(mlflow.pyfunc.PythonModel):
        def predict(self, context, model_input):
            from utils import my_func

            x = my_func(model_input)
            # .. your prediction logic
            return prediction


    # Log the model
    with mlflow.start_run() as run:
        mlflow.pyfunc.log_model(
            python_model=MyModel(),
            artifact_path="model",
            input_example=input_data,
            code_paths=["utils.py"],
        )

Then MLflow will save ``utils.py`` under ``code/`` directory in the model directory:

::

    model/
    ├── MLmodel
    ├── ...
    └── code/
        └── utils.py

When MLflow loads the model for serving, the ``code`` directory will be added to the system path so that you can use the module in your model
code like ``from utils import my_func``. You can also specify a directory path as ``code_path`` to save multiple files under the directory:

Caveats of ``code_path`` Option
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When using the ``code_path`` option, please be aware of the limitation that the specified file or directory **must be in the same directory as your model script**.
If the specified file or directory is in a parent or child directory like ``my_project/src/utils.py``, model serving will fail with ``ModuleNotFoundError``.
For example, let's say that you have the following file structure in your project

::

    my_project/
    |── train.py
    └── src/
        └──  utils.py

Then the following model code does **not** work:

.. code-block:: python

    class MyModel(mlflow.pyfunc.PythonModel):
        def predict(self, context, model_input):
            from src.utils import my_func

            # .. your prediction logic
            return prediction


    with mlflow.start_run() as run:
        mlflow.pyfunc.log_model(
            python_model=MyModel(),
            artifact_path="model",
            input_example=input_data,
            code_paths=[
                "src/utils.py"
            ],  # the file will be saved at code/utils.py not code/src/utils.py
        )

    # => Model serving will fail with ModuleNotFoundError: No module named 'src'

This limitation is due to how MLflow saves and loads the specified files and directories. When it copies the specified files or directories in ``code/`` target,
it does **not** preserve the relative paths that they were originally residing within. For instance, in the above example, MLflow will copy ``utils.py`` to ``code/utils.py``, not
``code/src/utils.py``. As a result, it has to be imported as ``from utils import my_func``, instead of ``from src.utils import my_func``.
However, this may not be pleasant, as the import path is different from the original training script.

To workaround this issue, the ``code_path`` should specify the parent directory, which is ``code_path=["src"]`` in this example.
This way, MLflow will copy the entire ``src/`` directory under ``code/`` and your model code will be able to import ``src.utils``.

.. code-block:: python

    class MyModel(mlflow.pyfunc.PythonModel):
        def predict(self, context, model_input):
            from src.utils import my_func

            # .. your prediction logic
            return prediction


    with mlflow.start_run() as run:
        mlflow.pyfunc.log_model(
            python_model=model,
            artifact_path="model",
            input_example=input_data,
            code_paths=["src"],  # the whole /src directory will be saved at code/src
        )

    # => This will work

.. warning::

    By the same reason, ``code_path`` option doesn't handle the relative import like ``code_path=["../src"]``.

Recommended Project Structure
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
With this limitation for ``code_path`` in mind, the recommended project structure looks like the following:

::

    my_project/
    |-- model.py # Defines the custom pyfunc model
    |── train.py # Trains and logs the model
    |── core/    # Required modules for prediction
    |   |── preprocessing.py
    |   └── ...
    └── helper/  # Other helper modules used for training, evaluation
        |── evaluation.py
        └── ...

Then you can log the model with ``code_paths=["core"]`` to include the required modules for prediction, while excluding the helper modules
that are only used for development.

.. _validating-environment-for-prediction:

Validating Environment for Prediction
-------------------------------------

Validating your model before deployment is a critical step to ensure production readiness.
MLflow provides a few ways to test your model locally, either in a virtual environment or a Docker container.
If you find any dependency issues during validation, please follow the guidance in :ref:`How to fix dependency errors when serving my model? <how-to-fix-dependency-errors-in-model>`

Testing offline prediction with a virtual environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
You can use MLflow Models **predict** API via Python or CLI to make test predictions with your model.
This will load your model from the model URI, create a virtual environment with the model dependencies (defined in MLflow Model),
and run offline predictions with the model.
Please refer to :py:func:`mlflow.models.predict()` or the `CLI reference <../cli.html#mlflow-models>`_ for more detailed usage for the predict API.

.. note::

    The Python API is available since MLflow 2.10.0. If you are using an older version, please use the CLI option.

.. tabs::

    .. code-tab:: python

        import mlflow

        mlflow.models.predict(
            model_uri="runs:/<run_id>/model",
            input_data=<input_data>,
        )

    .. code-tab:: bash

        mlflow models predict -m runs:/<run_id>/model-i <input_path>

Using the :py:func:`mlflow.models.predict()` API is convenient for testing your model and inference environment quickly.
However, it may not be a perfect simulation of the serving because it does not start the online inference server.

Testing online inference endpoint with a virtual environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
If you want to test your model by actually running the online inference server, you can use the  MLflow ``serve`` API.
This will create a virtual environment with your model and dependencies, similarly to the ``predict`` API, but will start the inference server
and expose the REST endpoints. Then you can send a test request and validate the response.
Please refer to the `CLI reference <../cli.html#mlflow-models>`_ for more detailed usage for the ``serve`` API.

.. code-block:: bash

    mlflow models serve -m runs:/<run_id>/model -p <port>
    # In another terminal
    curl -X POST -H "Content-Type: application/json" \
        --data '{"inputs": [[1, 2], [3, 4]]}' \
        http://localhost:<port>/invocations

While this is a reliable way to test your model before deployment, one caveat is that the virtual environment doesn't absorb the OS-level differences
between your machine and the production environment. For example, if you are using MacOS as a local dev machine but your deployment target is
running on Linux, you may encounter some issues that are not reproducible in the virtual environment.

In this case, you can use a Docker container to test your model. While it doesn't provide full OS-level isolation unlike virtual machines e.g. we
can't run Windows containers on Linux machines, Docker covers some popular test scenarios such as running different versions of Linux or simulating
Linux environments on Mac or Windows.

Testing online inference endpoint with a Docker container
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
MLflow ``build-docker`` API for CLI and Python is capable of building an Ubuntu-based Docker image for serving your model.
The image will contain your model and dependencies, as well as having an entrypoint that is used to start the inference server. Similarly to the `serve` API,
you can send a test request and validate the response.
Please refer to the `CLI reference <../cli.html#mlflow-models>`_ for more detailed usage for the ``build-docker`` API.

.. code-block:: bash

    mlflow models build-docker -m runs:/<run_id>/model -n <image_name>
    docker run -p <port>:8080 <image_name>
    # In another terminal
    curl -X POST -H "Content-Type: application/json" \
        --data '{"inputs": [[1, 2], [3, 4]]}' \
        http://localhost:<port>/invocations


.. _model-dependencies-troubleshooting:

Troubleshooting
---------------

.. _how-to-fix-dependency-errors-in-model:

How to fix dependency errors when serving my model?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
One of the most common issues experienced during model deployment centers around dependency issues. When logging or saving your model, MLflow tries to infer the
model dependencies and save them as part of the MLflow Model metadata. However, this might not always be complete and miss some dependencies e.g. [extras] dependencies
for certain libraries. This can cause errors when serving your model, such as "ModuleNotFoundError" or "ImportError". Below are some steps that can help to diagnose
and fix missing dependency errors.

.. hint::

    To reduce the possibility of dependency errors, you can add ``input_example`` when saving your model. This enables MLflow to 
    perform a model prediction before saving the model, thereby capturing the dependencies used during the prediction.
    Please refer to :ref:`Model Input Example <input-example>` for additional, detailed usage of this parameter.


1. Check the missing dependencies
*********************************
The missing dependencies are listed in the error message. For example, if you see the following error message:

.. code-block:: bash

    ModuleNotFoundError: No module named 'cv2'

2. Try adding the dependencies using the ``predict`` API
********************************************************
Now that you know the missing dependencies, you can create a new model version with the correct dependencies.
However, creating a new model for trying new dependencies might be a bit tedious, particularly because you may need to
iterate multiple times to find the correct solution. Instead, you can use the :py:func:`mlflow.models.predict()` API to test your change without
actually needing to re-log the model repeatedly while troubleshooting the installation errors.

To do so, use the **pip-requirements-override** option to specify pip dependencies like ``opencv-python==4.8.0``.

.. tabs::

    .. code-tab:: python

        import mlflow

        mlflow.models.predict(
            model_uri="runs:/<run_id>/model",
            input_data=<input_data>,
            pip_requirements_override=["opencv-python==4.8.0"],
        )

    .. code-tab:: bash

        mlflow models predict \
            -m runs:/<run_id>/model \
            -i <input_path> \
            --pip-requirements-override opencv-python==4.8.0

The specified dependencies will be installed to the virtual environment in addition to (or instead of) the dependencies
defined in the model metadata. Since this doesn't mutate the model, you can iterate quickly and safely to find the correct dependencies.

.. note::

    The ``pip-requirements-override`` option is available since MLflow 2.10.0.

3. Update the model metadata
****************************
Once you find the correct dependencies, you can create a new model with the correct dependencies.
To do so, specify the ``extra_pip_requirements`` option when logging the model.

.. code:: python

    import mlflow

    mlflow.pyfunc.log_model(
        artifact_path="model",
        python_model=python_model,
        extra_pip_requirements=["opencv-python==4.8.0"],
        input_example=input_data,
    )


How to migrate Anaconda Dependency for License Change
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Anaconda Inc. updated their `terms of service <https://www.anaconda.com/terms-of-service>`_ for anaconda.org channels. Based on the new terms of service you may require a commercial license if you rely on Anaconda’s packaging and distribution. See `Anaconda Commercial Edition FAQ <https://www.anaconda.com/blog/anaconda-commercial-edition-faq>`_ for more information. Your use of any Anaconda channels is governed by their terms of service.

MLflow models logged before `v1.18 <https://mlflow.org/news/2021/06/18/1.18.0-release/index.html>`_ were by default logged with the conda ``defaults`` channel (`https://repo.anaconda.com/pkgs/ <https://repo.anaconda.com/pkgs/>`_) as a dependency. Because of this license change, MLflow has stopped the use of the ``defaults`` channel for models logged using MLflow v1.18 and above. The default channel logged is now ``conda-forge``, which points at the community managed `https://conda-forge.org/ <https://conda-forge.org/>`_.

If you logged a model before MLflow v1.18 without excluding the ``defaults`` channel from the conda environment for the model, that model may have a dependency on the ``defaults`` channel that you may not have intended.
To manually confirm whether a model has this dependency, you can examine the ``channel`` value in the ``conda.yaml`` file that is packaged with the logged model. For example, a model's ``conda.yaml`` with a ``defaults`` channel dependency may look like this:

.. code-block:: yaml

    name: mlflow-env
    channels:
    - defaults
    dependencies:
    - python=3.8.8
    - pip
    - pip:
        - mlflow==2.3
        - scikit-learn==0.23.2
        - cloudpickle==1.6.0

If you would like to change the channel used in a model's environment, you can re-register the model to the model registry with a new ``conda.yaml``. You can do this by specifying the channel in the ``conda_env`` parameter of ``log_model()``.

For more information on the ``log_model()`` API, see the MLflow documentation for the model flavor you are working with, for example, :py:func:`mlflow.sklearn.log_model() <mlflow.sklearn.log_model>`.
