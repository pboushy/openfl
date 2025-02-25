.. # Copyright (C) 2020-2023 Intel Corporation
.. # SPDX-License-Identifier: Apache-2.0

.. _workflow_interface:

******************
Workflow Interface
******************

**Important Note**

The OpenFL workflow interface is experimental, subject to change, and is currently limited to single node execution. To setup and launch a real federation, see :ref:`running_a_federation`

What is it?
===========

A new OpenFL interface that gives significantly more flexility to researchers in the construction of federated learning experiments. It is heavily influenced by the interface and design of `Metaflow` , the popular framework for data scientists originally developed at Netflix. There are several reasons we converged on Metaflow as inspiration for our work:

1. Clean expression of task sequence. Flows start with a `start` task, and end with `end`. The next task in the sequence is called by `self.next`.
2. Easy selection of what should be sent between tasks using `include` or `exclude`
3. Excellent tooling ecosystem: the metaflow client gives easy access to prior runs, tasks, and data artifacts generated by an experiment. 

There are several modifications we make in our reimagined version of this interface that are necessary for federated learning:

1. *Placement*: Metaflow's `@step` decorator is replaced by placement decorators that specify where a task will run. In horizontal federated learning, there are server (or aggregator) and client (or collaborator) nodes. Tasks decorated by `@aggregator` will run on the aggregator node, and `@collaborator` will run on the collaborator node. These placement decorators are interpreted by *Runtime* implementations: these do the heavy lifting of figuring out how to get the state of the current task to another process or node. 
2. *Runtime*: Each flow has a `.runtime` attribute. The runtime encapsulates the details of the infrastucture where the flow will run. In this experimental release, we support only a `LocalRuntime` single node implementation, but as this work matures, we will extend to a `FederatedRuntime` that implements distributed operation across remote infrastructure.
3. *Conditional branches*: Perform different tasks if a criteria is met
4. *Loops*: Internal loops are within a flow; this is necessary to support rounds of training where the same sequence of tasks is performed repeatedly.   

How to use it?
==============

Let's start with the basics. A flow is intended to define the entirety of federated learning experiment. Every flow begins with the `start` task and concludes with the `end` task. At each step in the flow, attributes can be defined, modified, or deleted. Attributes get passed forward to the next step in the flow, which is defined by the name of the task passed to the `next` function. In the line before each task, there is a **placement decorator**. The placement decorator defines where that task will be run. The OpenFL Workflow Interface adopts the conventions set by Metaflow, that every workflow begins with start and concludes with the end task. In the following example, the aggregator begins with an optionally passed in model and optimizer. The aggregator begins the flow with the start task, where the list of collaborators is extracted from the runtime (:code:`self.collaborators = self.runtime.collaborators`) and is then used as the list of participants to run the task listed in self.next, aggregated_model_validation. The model, optimizer, and anything that is not explicitly excluded from the next function will be passed from the start function on the aggregator to the aggregated_model_validation task on the collaborator. Where the tasks run is determined by the placement decorator that precedes each task definition (:code:`@aggregator` or :code:`@collaborator`). Once each of the collaborators (defined in the runtime) complete the aggregated_model_validation task, they pass their current state onto the train task, from train to local_model_validation, and then finally to join at the aggregator. It is in join that an average is taken of the model weights, and the next round can begin. 

.. code-block:: python

    class FederatedFlow(FLSpec):

        def __init__(self, model = None, optimizer = None, rounds=3, **kwargs):
            super().__init__(**kwargs)
            if model is not None:
                self.model = model
                self.optimizer = optimizer
            else:
                self.model = Net()
                self.optimizer = optim.SGD(self.model.parameters(), lr=learning_rate,
                                       momentum=momentum)
            self.rounds = rounds

        @aggregator
        def start(self):
            print(f'Performing initialization for model')
            self.collaborators = self.runtime.collaborators
            self.private = 10
            self.current_round = 0
            self.next(self.aggregated_model_validation,foreach='collaborators',exclude=['private'])

        @collaborator
        def aggregated_model_validation(self):
            print(f'Performing aggregated model validation for collaborator {self.input}')
            self.agg_validation_score = inference(self.model,self.test_loader)
            print(f'{self.input} value of {self.agg_validation_score}')
            self.next(self.train)

        @collaborator
        def train(self):
            self.model.train()
            self.optimizer = optim.SGD(self.model.parameters(), lr=learning_rate,
                                       momentum=momentum)
            train_losses = []
            for batch_idx, (data, target) in enumerate(self.train_loader):
              self.optimizer.zero_grad()
              output = self.model(data)
              loss = F.nll_loss(output, target)
              loss.backward()
              self.optimizer.step()
              if batch_idx % log_interval == 0:
                print('Train Epoch: 1 [{}/{} ({:.0f}%)]\tLoss: {:.6f}'.format(
                   batch_idx * len(data), len(self.train_loader.dataset),
                  100. * batch_idx / len(self.train_loader), loss.item()))
                self.loss = loss.item()
                torch.save(self.model.state_dict(), 'model.pth')
                torch.save(self.optimizer.state_dict(), 'optimizer.pth')
            self.training_completed = True
            self.next(self.local_model_validation)

        @collaborator
        def local_model_validation(self):
            self.local_validation_score = inference(self.model,self.test_loader)
            print(f'Doing local model validation for collaborator {self.input}: {self.local_validation_score}')
            self.next(self.join, exclude=['training_completed'])

        @aggregator
        def join(self,inputs):
            self.average_loss = sum(input.loss for input in inputs)/len(inputs)
            self.aggregated_model_accuracy = sum(input.agg_validation_score for input in inputs)/len(inputs)
            self.local_model_accuracy = sum(input.local_validation_score for input in inputs)/len(inputs)
            print(f'Average aggregated model validation values = {self.aggregated_model_accuracy}')
            print(f'Average training loss = {self.average_loss}')
            print(f'Average local model validation values = {self.local_model_accuracy}')
            self.model = FedAvg([input.model for input in inputs])
            self.optimizer = [input.optimizer for input in inputs][0]
            self.current_round += 1
            if self.current_round < self.rounds:
                self.next(self.aggregated_model_validation, foreach='collaborators', exclude=['private'])
            else:
                self.next(self.end)

        @aggregator
        def end(self):
            print(f'This is the end of the flow')  


Background
==========

Prior interfaces in OpenFL support the standard horizontal FL training workflow:

    1. The collaborator downloads the latest model from the aggregator
    2. The collaborator performs validation with their local validation dataset on the aggregated model, and sends these metrics to the aggregator (aggregated_model_validation task)
    3. The collaborator trains the model on their local training data set, and sends the local model weights and metrics to the aggregator (train task)
    4. The collaborator performs validation with their local validation dataset on their locally trained model, and sends their validation metrics to the aggregator (locally_tuned_model_validation task)
    5. The aggregator applies an aggregation function (weighted average, FedCurv, FedProx, etc.) to the model weights, and reports the aggregate metrics.

The Task Assigner determines the list of collaborator tasks to be performed, and both in the task runner API as well as the interactive API these tasks can be modified (to varying degrees). For example, to perform federated evaluation of a model, only the aggregated_model_validation task would be selected for the assigner's block of the federated plan. Equivalently for the interactive API, this can be done by only registering a single validation task. But there are many other types of workflows that can't be easily represented purely by training / validation tasks performed on a collaborator with a single model. An example is training a Federated Generative Adversarial Network (GAN); because this may be represented by separate generative and discriminator models, and could leak information about a collaborator dataset, the interface we provide should allow for better control over what gets sent over the network and how. Another common request we get is for validation with an aggregator's dataset after training. Prior to |productName| 1.5, there has not a great way to support this in OpenFL.

Goals
=====

    1. Simplify the federated workflow representation
    2. Clean separation of workflow from runtime infrastructure
    3. Help users better understand the steps in federated learning (weight extraction, tensor compression, etc.)
    4. Interface makes it clear what is sent across the network
    5. The placement of tasks and how they connect should be straightforward
    6. Don't reinvent unless absolutely necessary

Workflow Interface API
======================

The workflow interface formulates the experiment as a series of tasks, or a flow. Every flow begins with the `start` task and concludes with `end`.

Runtimes
========

A :code:`Runtime` defines where the flow will be executed, who the participants are in the experiment, and the private information that each participant has access to. In this experimental release, single node execution is supported using the :code:`LocalRuntime`. Let's see how a :code:`LocalRuntime` is created:

.. code-block:: python
    
    # Aggregator
    aggregator_ = Aggregator()

    collaborator_names = ["Portland", "Seattle", "Chandler", "Bangalore"]

    def callable_to_initialize_collaborator_private_attributes(index, n_collaborators, batch_size, train_dataset, test_dataset):
        train = deepcopy(train_dataset)
        test = deepcopy(test_dataset)
        train.data = train_dataset.data[index::n_collaborators]
        train.targets = train_dataset.targets[index::n_collaborators]
        test.data = test_dataset.data[index::n_collaborators]
        test.targets = test_dataset.targets[index::n_collaborators]

        return {
            "train_loader": torch.utils.data.DataLoader(train, batch_size=batch_size, shuffle=True),
            "test_loader": torch.utils.data.DataLoader(test, batch_size=batch_size, shuffle=True),
        }

    # Setup collaborators private attributes via callable function
    collaborators = []
    for idx, collaborator_name in enumerate(collaborator_names):
        collaborators.append(
            Collaborator(
                name=collaborator_name,
                private_attributes_callable=callable_to_initialize_collaborator_private_attributes,
                index=idx, 
                n_collaborators=len(collaborator_names),
                train_dataset=mnist_train, 
                test_dataset=mnist_test, 
                batch_size=64
            )
        )

    local_runtime = LocalRuntime(aggregator=aggregator_, collaborators=collaborators)

Let's break this down, starting with the :code:`Aggregator` and :code:`Collaborator` components. These components represent the *Participants* in a Federated Learning experiment. Each participant has its own set of *private attributes* that represent the information / data specific to its role or requirements. As the name suggests these *private attributes* are accessible only to the particular participant, and are appropriately inserted into or filtered out of current Flow state when transferring from between Participants. For e.g. Collaborator private attributes are inserted into :code:`flow` when transitioning from Aggregator to Collaborator and are filtered out when transitioning from Collaborator to Aggregator.

In the above :code:`FederatedFlow`, each collaborator accesses train and test datasets via *private attributes* :code:`train_loader` and :code:`test_loader`. These *private attributes* need to be set using a (user defined) callback function while instantiating the participant. Participant *private attributes* are returned by the callback function in form of a dictionary, where the key is the name of the attribute and the value is the object.

In this example callback function :code:`callable_to_initialize_collaborator_private_attributes()` returns the collaborator private attributes :code:`train_loader` and :code:`test_loader` that are accessed by collaborator steps (:code:`aggregated_model_validation`, :code:`train` and :code:`local_model_validation`). Some important points to remember while creating callback function and private attributes are: 

   - Callback Function needs to  be defined by the user and should return the *private attributes* required by the participant in form of a key/value pair 
   - In above example multiple collaborators have the same callback function. Depending on the Federated Learning requirements, user can specify unique callback functions for each Participant
   - If no Callback Function is specified then the Participant shall not have any *private attributes*
   - Callback function can be provided with any parameters required as arguments. In this example, parameters essential for the callback function are supplied with corresponding values bearing *same names* during the instantiation of the Collaborator

        * :code:`index`: Index of the particular collaborator needed to shard the dataset
        * :code:`n_collaborators`: Total number of collaborators in which the dataset is sharded
        * :code:`batch_size`: For the train and test loaders
        * :code:`train_dataset`: Train Dataset to be sharded between n_collaborators 
        * :code:`test_dataset`: Test Dataset to be sharded between n_collaborators
          
   - Callback function needs to be specified by user while instantiating the participant. Callback function is invoked by the OpenFL runtime at the time participant is created and once created these attributes cannot be modified
   - Private attributes are accessible only in the Participant steps

Now let's see how the runtime for a flow is assigned, and the flow gets run:

.. code-block:: python
   
    flow = FederatedFlow()
    flow.runtime = local_runtime
    flow.run()
    
And that's it! This will run an instance of the :code:`FederatedFlow` on a single node in a single process. 

Runtime Backends
================

The Runtime defines where code will run, but the Runtime has a :code:`Backend` - which defines the underlying implementation of *how* the flow will be executed. :code:`single_process` is the default in the :code:`LocalRuntime`: it executes all code sequentially within a single python process, and is well suited to run both on high spec and low spec hardware

For users with large servers or multiple GPUs they wish to take advantage of, we also provide a :code:`ray` `<https://github.com/ray-project/ray>` backend. The Ray backend enables parallel task execution for collaborators, and optionally allows users to request dedicated CPU / GPUs for Participants by using the :code:`num_cpus` and :code:`num_gpus` arguments while instantiating the Participant in following manner:

.. code-block:: python
    
    # Aggregator
    aggregator_ = Aggregator(num_gpus=0.2)

    collaborator_names = ["Portland", "Seattle", "Chandler", "Bangalore"]

    def callable_to_initialize_collaborator_private_attributes(index, n_collaborators, batch_size, train_dataset, test_dataset):
        ... 
        
    # Setup collaborators private attributes via callable function
    collaborators = []
    for idx, collaborator_name in enumerate(collaborator_names):
        collaborators.append(
            Collaborator(
                name=collaborator_name,
                num_gpus=0.2, # Number of the GPU allocated to Participant
                private_attributes_callable=callable_to_initialize_collaborator_private_attributes,
                index=idx, 
                n_collaborators=len(collaborator_names),
                train_dataset=mnist_train, 
                test_dataset=mnist_test, 
                batch_size=64
            )
        )

     # The Ray Backend will now be used for local execution
     local_runtime = LocalRuntime(aggregator=aggregator, collaborators=collaborators, backend='ray')

In the above example, we have used :code:`num_gpus=0.2` while instantiating Aggregator and Collaborator to specify that each participant shall use 1/5th of GPU - this results in one GPU being dedicated for a total of 4 collaborators and 1 Aggregator. Users can tune these arguments based on their Federated Learning requirements and available hardware resources. Configurations where one Participant is shared across GPUs is not supported. For e.g. trying to run 5 participants on 2 GPU hardware with :code:`num_gpus=0.4` will not work since 80% of each GPU is allocated to 4 participants and 5th participant does not have any available GPU remaining for use.

**Note:** It is not necessary to have ALL the participants use GPUs. For e.g. only the Collaborator are allocated to GPUs. In this scenario user should ensure that the artifacts returned by Collaborators to Aggregator (e.g. locally trained model object) should be loaded back to CPU before exiting the collaborator step (i.e. before the join step). As Tensorflow manages the object allocation by default therefore this step is needed only for Pytorch.

Debugging with the Metaflow Client
==================================

Federated learning is difficult to debug. A common example of this difficulty comes in the form of mislabeled datasets. Even one mislabeled dataset on a collaborator's training set in a large federation can result model convergence delay and lower aggregate accuracy. Wouldn't it be better to pinpoint these problems early instead of after the full experiment has taken place?

To improve debugging of federated learning experiments, we are reusing Metaflow's interfaces to (optionally) save all of the attributes generated by each participant, every task's stdout / stderr, and provide a visual representation of the workflow graph. 

Capturing this information requires just a one line change to the Flow object initialization by setting :code:`checkpoint=True`:

.. code-block:: python
    
   flow = FederatedFlow(..., checkpoint=True)
   
After the flow has started running, you can use the Metaflow Client to get intermediate information from any of the participants tasks:

.. code-block:: python
    
   from metaflow import Metaflow, Flow, Step, Task

   # Initialize Metaflow object and obtain list of executed flows:
   m = Metaflow()
   list(m)
   > [Flow('FederatedFlow'), Flow('AggregatorValidationFlow'), Flow('FederatedFlow_MNIST_Watermarking')]

   # The name of the flow is the name of the class
   # Identify the Flow name
   flow_name = 'FederatedFlow'

   # List all instances of Federatedflow executed under distinct run IDs
   flow = Flow(flow_name)
   list(flow)
   > [Run('FederatedFlow/1692946840822001'),
      Run('FederatedFlow/1692946796234386'),
      Run('FederatedFlow/1692902602941163'),
      Run('FederatedFlow/1692902559123920'),]

   # To Retrieve the latest run of the Federatedflow
   run = Flow(flow_name).latest_run
   print(run)
   > Run('FederatedFlow/1692946840822001')

   list(run)
   > [Step('FederatedFlow/1692946840822001/end'),
      Step('FederatedFlow/1692946840822001/join'),
      Step('FederatedFlow/1692946840822001/local_model_validation'),
      Step('FederatedFlow/1692946840822001/train'),
      Step('FederatedFlow/1692946840822001/aggregated_model_validation'),
      Step('FederatedFlow/1692946840822001/start')]
   step = Step('FederatedFlow/1692946840822001/aggregated_model_validation')
   for task in step:
       if task.data.input == 'Portland':
           print(task.data)
           portland_task = task
           model = task.data.model
   > <MetaflowData: train_loader, collaborators, loss, optimizer, model, input, rounds, agg_validation_score, current_round, test_loader, training_completed>
   print(model)
   > Net(
      (conv1): Conv2d(1, 10, kernel_size=(5, 5), stride=(1, 1))
      (conv2): Conv2d(10, 20, kernel_size=(5, 5), stride=(1, 1))
      (conv2_drop): Dropout2d(p=0.5, inplace=False)
      (fc1): Linear(in_features=320, out_features=50, bias=True)
      (fc2): Linear(in_features=50, out_features=10, bias=True)
    )

And if we wanted to get log or error message for that task, you can just run:

.. code-block:: python
    
   print(portland_task.stdout)
   > Train Epoch: 1 [0/15000 (0%)]	Loss: 2.295608
     Train Epoch: 1 [640/15000 (4%)]	Loss: 2.311402
     Train Epoch: 1 [1280/15000 (9%)]	Loss: 2.281983
     Train Epoch: 1 [1920/15000 (13%)]	Loss: 2.269565
     Train Epoch: 1 [2560/15000 (17%)]	Loss: 2.261440
     ...
   print(portland_task.stderr)
   > [No output]

Also, If we wanted to get the best model and the last model, you can just run:

.. code-block:: python

    # Choose the specific step containing the desired models (e.g., 'join' step):
    step = Step('FederatedFlow/1692946840822001/join')
    list(step)
    > [Task('FederatedFlow/1692946840822001/join/12'),--> Round 3
       Task('FederatedFlow/1692946840822001/join/9'), --> Round 2
       Task('FederatedFlow/1692946840822001/join/6'), --> Round 1
       Task('FederatedFlow/1692946840822001/join/3')] --> Round 0

    """The sequence of tasks represents each round, with the most recent task corresponding to the final round and the preceding tasks indicating the previous rounds 
    in chronological order.
    To determine the best model, analyze the command line logs and model accuracy for each round. Then, provide the corresponding task ID associated with that Task"""
    task = Task('FederatedFlow/1692946840822001/join/9')

    # Access the best model and its associated data
    best_model = task.data.model
    best_local_model_accuracy = task.data.local_model_accuracy
    best_aggregated_model_accuracy = t.data.aggregated_model_accuracy

    # To retrieve the last model, select the most recent Task i.e last round.
    task = Task('FederatedFlow/1692946840822001/join/12')
    last_model = task.data.model

    # Save the chosen models using a suitable framework (e.g., PyTorch in this example):
    import torch
    torch.save(last_model.state_dict(), PATH)
    torch.save(best_model.state_dict(), PATH)

While this information is useful for debugging, depending on your workflow it may require significant disk space. For this reason, `checkpoint` is disabled by default.

Runtimes: Future Plans
======================

Our goal is to make it a one line change to configure where and how a flow is executed. While we only support single node execution with the :code:`LocalRuntime` today, our aim in future releases is to make going from one to multiple nodes as easy as:

.. code-block:: python
   
    flow = FederatedFlow()
    # Run on a single node first
    local_runtime = LocalRuntime(aggregator=aggregator, collaborators=collaborators)
    flow.runtime = local_runtime
    flow.run()
    
    # A future example of how the same flow could be run on distributed infrastructure
    federated_runtime = FederatedRuntime(...)
    flow.runtime = federated_runtime
    flow.run()