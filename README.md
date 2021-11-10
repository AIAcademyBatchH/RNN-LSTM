# RNN-LSTM

Datasets:
https://drive.google.com/drive/folders/15utTiEZVGztXzJZle2qV84YeyJUvisJs?usp=sharing


q='''

I am new to luigi, came across it while designing a pipeline for our ML efforts and though it wasn't fitted to my particular use case it had so many extra features I decided to fit it to my use case, basically what I was looking for was a way to be able to persist a custom built pipeline and thus have its result repeatable, after reading most of the online tutorials I tried to implement my serialization using the existing luigi.cfg configuration and command line mechanisms and it might have sufficed for the tasks' parameters but it provided no way of serializing the DAG connectivity of my pipeline, so I decided to have a WrapperTask which received a json config file which would create all the task instances and connect all the plumbing. I hereby enclose a small test program for your scrutiny:

import random
import luigi
import time
import os


class TaskNode(luigi.Task):
    i = luigi.IntParameter()  # node ID

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.required = []

    def set_required(self, required=None):
        self.required = required  # set the dependencies
        return self

    def requires(self):
        return self.required

    def output(self):
        return luigi.LocalTarget('{0}{1}.txt'.format(self.__class__.__name__, self.i))

    def run(self):
        with self.output().open('w') as outfile:
            outfile.write('inside {0}{1}\n'.format(self.__class__.__name__, self.i))
        self.process()

    def process(self):
        raise NotImplementedError(self.__class__.__name__ + " must implement this method")


class FastNode(TaskNode):

    def process(self):
        time.sleep(1)


class SlowNode(TaskNode):

    def process(self):
        time.sleep(2)


# This WrapperTask builds all the nodes 
class All(luigi.WrapperTask):

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        num_nodes = 513

        classes = TaskNode.__subclasses__()
        self.nodes = []
        for i in reversed(range(num_nodes)):
            cls = random.choice(classes)

            dependencies = random.sample(self.nodes, (num_nodes - i) // 35)

            obj = cls(i=i)
            if dependencies:
                obj.set_required(required=dependencies)
            else:
                obj.set_required(required=None)

            # delete existing output causing a build all
            if obj.output().exists():
                obj.output().remove()  

            self.nodes.append(obj)

    def requires(self):
        return self.nodes


if __name__ == '__main__':
    luigi.run()
So, basically, as states in the question title, this focuses on the dynamic dependencies and generates a 513 node dependency DAG with p=1/35 connectivity probability, it also makes the All WrapperTask class require all nodes (I have a version which only connects it to heads of connected DAG components but I didn't want to over complicate).

Is there a more standard way of implementing this? Especially note the not so pretty complication with the TaskNode init and set_required methods, I only did it this way because receiving parameters in the init method clashes somehow with the way luigi registers parameters. I also tried several other ways but this was basically the most decent one (that worked)

If there isn't a standard way I'd still love to hear any insights you have on the way I plan to go before I implement the entire piping framework.'''
