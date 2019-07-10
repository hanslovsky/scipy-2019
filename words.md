# Title Slide

Good morning,

I am Philipp Hanslovsky from the Saalfeld lab at HHMI Janelia and today I will introduce imglyb, a Python package that bridges the chasm between two scientific image processing frameworks that are widely used in many fields of research: Python with SciPy and NumPy on the one hand, Java with ImageJ and ImgLib2 on the other hand. When I moved from Heidelberg, Germany to the DC metro area about five years ago I did not know that I had my biggest journey still ahead of me, a journey not through space but from one programming language and framework to another.

# Blank Mountains

I first started programming seriously during my Master's with the Hamprecht lab in Heidelberg. The lab uses C++ and Python to create high-performance software for bio-image analysis, e.g. neuron reconstruction (segmentation) or cell tracking. Naturally, I was exposed to C++ and Python and after hard work and learning a lot of concepts I was finally able to reach the peak of Mount SciPy with all the great software and frameworks like NumPy, scikit-image, or Matplotlib, and many more. 

# SciPy Mountain

I became very comfortable with arithmetic ndarray operations and operator overloading and the concise and readable code that can be created with the SciPy framework and I could not imagine that anyone would use anything else for image processing.

Little did I know that there was a whole other mountain: After my arrival in the US, I sadly realized that I had to come down from Mt SciPy and climb another peak instead: Mount ImageJ and the Java programming language. The ascent was hard because I had to forget concepts that I had learned on my way up Mount SciPy and too many times I fell back to those patterns that are unfortunately not very efficient on this mountain. 

# ImageJ Mountain

After some time I reached the peak of this mountain as well and all the hard work had paid off, eventually. The ImageJ distribution Fiji provides a whole universe of tools that you may not know but are very important for the day-to-day work of many researchers in the Bioimage community. The multi-dimensional generic image processing library ImgLib2 at the core of modern ImageJ2 virtualizes pixel access for generic and re-usable algorithms that are agnostic of the underlying data: Data can be loaded from disk, streamed from the internet, or be entirely virtual, e.g. a mathematical function evaluated at the pixel location. I am mostly interested in powerful visualization toos like BigDataViewer developed by Tobias Pietzsch. BigDataViewer is a re-slicing viewer that renders two-dimensional cross-sections through volumetric data at arbitrary orientations and zoom levels and domain specific tools build on top of it, for example, MaMuT for visualization, tracking and lineaging or my own tool Paintera for efficient generation of dense volumetric ground truth annotations and proof reading neuron segmentation.

# BDV Video

If you are in a similar position as I was a little more than five years ago, you will probably need some convincing that the ImageJ world is worth exploring. Let me show a short video of BigDataViewer usage to spark your interest in the ImageJ world. This example demonstrates free navigation of a 100 Teravoxel source, FAFB, the first ever electron microscopy acquisition of a complete adult Drosophila brain. Multiple sources can be composed, in this case synapse predictions for the entire dataset by Larissa are rendered in magenta over the raw data. We can scroll through the data, rotate freely, change the zoom level. The contrast range of each source can be adjusted independently. Now we disable the raw data to render only the synapse predictions and zoom out as much as possible and the synapse predictions on EM appear similar to light microscopy with NC82 synapse labeling, in particular if the color mapping is changed to green. Now we turn back on the raw data and zoom back in to confirm that the synapse predictions are correct.

# Back To Slide

Unfortunately, a deep chasm separates the worlds of SciPy and ImageJ: Combining both frameworks in a single workflow is cumbersome at best. Shared memory access is not possible and intermediate results have to be written to and read from disk, which is prohibitive for high performance computation such as visualization of large datasets in BigDataViewer.

If you have any experience with Python wrappers for natively compiled languages like C or C++, you would probably tell me: It is pretty easy to get there, just expose your data structures as NumPy arrays and any functions that you need to Python. Unfortunately, for Java that is not as straight forward:

# What is Python

Python is an interpreted language with dynamic typing. The original C-implementation, CPython, can access native memory and can be extended through C/C++ extensions, including Cython. The Python software stack includes interactive shells and notebooks that have become very popular among scientists.

# What is Java

Java on the other hand is a statically typed language that compiles into byte code. The byte code is executed within the Java Virtual Machine (JVM) that adds a layer of abstraction on top of the operating system, including memory. As a consequence, native memory cannot be accessed through the Java language API. There are, however, ways to access native memory in Java: The Java Native Interface or through internal classes of the Java Runtime Environment. These options are rarely used because they break the memory abstraction introduced by the JVM, including garbage collection.

# Why are they hard to combine?

In summary, Java runs in a virtual machine with an additional layer of abstraction of memory, and Python accesses native memory. A non-obvious consequence is that Java arrays are not the same as C arrays, even though nomenclature and syntax are similar. In particular this means, that a Java image processing library that does not virtualize pixel access and is hard coded to use Java arrays will never be able to share memory with NumPy arrays. 

So how can we get around this and make SciPy and ImageJ talk to each other?

# Ways out of the dilemma 1

One possible solution is inter-process communication: Two independent processes, one for Python and one for Java, talk to each other through sockets. 

# Ways out of the dilemma 2

This is comparatively easy and can be extended to any language, i.e. once a protocol is defined, it has to be implemented only once per language and not per each pair of languages.

The initial design goal was to provide shared memory access between Java and Python and this is not possible with inter-process communication: Unnecessary copies of data are sent around between processes, memory cannot be modified in place, if desired.

# Ways out of the dilemma 3

Alternatively, Python and the JVM in the same single process make shared memory possible: 

# Ways out of the dilemma 4
data can be modified in place and unnecessary copies of data are avoided. Even though the Java language API cannot access native memory, the Java Native Interface and internal methods of the Java runtime can be used to read and write from native memory.

# Two options 1

In general, there are two ways to achieve this goal: One option is to use the Java implementation of Python: Jython. 

# Two options 2

Native CPython extensions can be accessed through the Jython Native Interface, similar to the Java Native Interface. 

# Two options 3

The Jython version, however, is outdated and only Jython 2.7 is available ([Python 3 statement](https://python3statement.org)!!). Work on Jython 3 seems to have stalled.

# Two options 4

The other approach is to start Java within a CPython process and this is the path that I followed.

This involves the following steps:

# Way out! 1 

First, a JVM needs to be launched from within a CPython process. A compatibility layer must expose Java classes and methods to the Python interpreter and make them accessible to the user. I used the existing PyJNIus library of the kivy framework for this compatibility layer.

# Way out! 2

Second, ImgLib2 and NumPy need to be able to understand each other's data. Only the clear separation of concerns of pixel access and data storage in ImgLib2 makes shared memory between NumPy and ImgLib2 possible. 

First, I created 

# Way out! 3
 - *imglib2-unsafe* to read from and write into native memory through ImgLib2
 
# Way out! 4
 - *imglib2-imglyb* ImgLib2 data structures and wrappers that understand NumPy meta data such as shape, strides, or offsets into native memory
 
# Way out! 5
 - *imglyb* the Python interface that easily translates numpy arrays into ImgLib2 data structures with shared memory access

This native memory access is only possible because of the virtualization of pixel access in ImgLib2: algorithms processing ImgLib2 data structures are generally not aware if the data is provided as Java array or native array.

# imglyb 1
Going back to the initial metaphor of two peaks that are separated by a deep chasm: 

# imglyb 2
With imglyb (with a y), I built a bridge over that chasm to connect the SciPy and ImageJ peaks and make interoperability easier and more efficient through shared memory.

# Installation & Usage 1
I hope that you all are psyched now and ready to install imglyb because it is really easy to get: The most simple way is to install imglyb from conda. I made imglyb and its PyJNIus dependency available on conda-forge to make sure that it will be available and up to date in the future.

# Installation & Usage 2 and 3
Examples are available on PyPI and I created notebooks for more interactive demonstration. I will show some of those notebooks in a few minutes.

# How to use?
How can we use imglyb? It is as simple as `import imglyb`. It is important to note that `imglyb` must be imported before any `jnius` import with the exception of `jnius_config`.

# Wrap NumPy arrays in ImgLib2
For example, these lines will wrap and display 3D `numpy.ndarray`s: First, import `imglyb` and `numpy`, then create some random data with numpy, wrap it, and display it with BigDataViewer. A second uint32 image is wrapped as ARGB image and displayed in BigDataViewer as well.

# Examples
More complex examples are available in the `imglyb-examples` package:

# Known Issues
Before I demonstrate imglyb use in interactive notebooks, I will talk briefly about the current limitations: Currently, graphical user interfaces built upon the Java awt or Swing frameworks, e.g. BigDataViewer, require a wrapper script on macOS. This wrapper script is installed with imglyb. Dask arrays cannot be wrapped as ImgLib2 data structures  yet.

In the following I will go through four short example notebooks to demonstrate how to use `imglyb`:

# Example 1 - BDV
Example 1: I will open a HDF5 file with the `h5` package and display the NumPy array in BigDataViewer. BigDataViewer is a Java viewer for arbitrarily large multi-view 3D images and 3D image sequences and developed by Tobias Pietzsch. Show Example!

# Example 2 - BDV
Example 2: I will implement an extension for BigDataViewer with which I can paint into a 3D NumPy array.

# Example 3 - BigWarp
Example 3: BigWarp is a tool for interactive alignment of 2D or 3D images through pairs of manually placed landmarks. This is used, for example, to align micrographs of different modalities, light and EM, of different specimen of the same animal species. I will demonstrate how to warp a 2D NumPy array with BigWarp.

# Example 4 - Paintera
Example 4: Paintera is a tool for the efficient generation of dense ground truth annotations nad proof-reading 3D EM connectomics with 3D visualization and mesh generation on the fly. I will demonstrate how to use Paintera to visualize NumPy label data without the need to pre-compute triangle meshes.

This brings me to the end of my talk.

# Resources
I used these logos and cliparts.

# Thank you
Finally, I would like to thank you all for attending this talk and I hope that you all are excited that you can share memory between SciPy and ImageJ/ImgLib2 now! As a reminder, I placed the installation instructions and important links on this last slide. The slides are available online at the last link. I will leave Saturday afternoon but should be around for the sprints on Saturday morning to help setup `imglyb` or to give advice for hacking `imglyb` if people are interested.

Thank you for your attention and please feel free to ask questions now or talk to me in person during the conference. 

