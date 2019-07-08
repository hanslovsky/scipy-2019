Good morning,

I am Philipp Hanslovsky from the Saalfeld lab at HHMI Janelia and today I will introduce imglyb, a Python package that bridges the chasm between two scientific and image processing frameworks that are widely used in many fields of research: Python with SciPy and NumPy on the one hand, Java with ImageJ and ImgLib2 on the other hand.

When I moved from Heidelberg, Germany to the DC metro area about five years ago I did not know that I had my biggest journey still ahead of me: I first started programming seriously during my Master's with the Hamprecht lab in Heidelberg. The lab uses C++ and Python to create high-performance software for bio-image analysis, e.g. neuron reconstruction (segmentation) or cell tracking. Naturally, I was exposed to C++ and Python and after hard work and learning a lot of concepts I was finally able to reach the peak of Mount SciPy with all the great software and frameworks like NumPy, scikit-image, or Matplotlib, and many more. I became very comfortable with arithmetic ndarray operations and operator overloading and the concise and readable code that can be created with the SciPy framework and I could not imagine that anyone would use anything else for image processing.

Little did I know that there was a whole other mountain: After my arrival in the US, I sadly realized that I had to come down from Mt SciPy and climb another peak instead: Mount ImageJ and the Java programming language. The ascent was hard because I had to forget concepts that I had learned on my way up Mount SciPy and too many times I fell back to those patterns that are unfortunately not very efficient on this mountain. After some time I reached the peak of this mountain as well and all the hard work had paid off, eventually: I finally understood the design of the core library ImgLib2 and its powerful virtualization of pixel access allow for very generic and re-usable algorithms that can be entirely agnostic of the underlying data: Data can be loaded from disk, streamed from the internet, or be entirely virtual, e.g. a mathematical function evaluated at the pixel location. Just like on Mount SciPy, a vast amount of open source software is available here: ImgLib2 is the core of modern ImageJ2 and other Software like Paintera, the Fiji distribution of ImageJ supplies researchers with a vast number of plugins and extensions, MaMuT is used for visualization, tracking, and lineaging in large multi-view images.

I found myself looking over to Mount SciPy that seemed so close and even though I was very comfortable and appreciated all the perks here on Mount ImageJ, I still was frustrated that I was unable to use the SciPy framework anymore. I would have to descent from Mount ImageJ and climb Mount SciPy again because the two peaks were separated by a deep chasm. And while subsecent ascents are always easier than the first ascent, I remained frustrated about the lack of interoperability, i.e. in-process shared memory data structures. Combining both frameworks in a single workflow is cumbersome, at best: Intermediate results have to be written to disk and then loaded in the other framework, data writing and loading may not be compatible, etc. 

If you have any experience with Python wrappers for natively compiled languages like C or C++, you would probably tell me: It is pretty easy to get there, just expose your data structures as NumPy arrays and any functions that you need to Python. Unfortunately, for Java that is not as straight forward:

Python is an interpreted language with dynamic typing. The original C-implementation, CPython can access native memory and can be extended through C/C++ extensions, including Cython. The Python software stack includes interactive shells and notebooks that have become very popular among scientists.

Java on the other hand is a statically typed language that compiles into byte code. The byte code is executed within a virtual machine (JVM) that adds a layer of abstraction on top of the operating system: Java code compiled on my machine will run on any other computer with a compatible Java Runtime and software distribution is a lot easier compared to native software. As a consequence, native memory cannot be accessed through the Java language API, and this is the reason why Java and Python are hard to combine:

Java runs in a virtual machine with an additional layer of abstraction of memory, and Python accesses native memory. A non-obvious consequence is that Java arrays are not the same as C arrays, even though nomenclature and syntax are similar. So how can we get around this and make SciPy and ImageJ talk to each other?

One possible solution is inter-process communication: Two independent processes, one for Python and one for Java, talk to each other through sockets. This is comparatively easy and can be extended to any language, i.e. once a protocol is defined, it has to be implemented only once per language and not per each pair of languages. Onthe downside, memory cannot be shared between Java and Python with this approach: Unnecessary copies of data are sent around between processes, memory cannot be modified in place, if desired.

The alternative of Python and the JVM in the same process makes shared memory possible: data can be modified in place and unnecessary copies of data are avoided. Even though the Java language API cannot access native memory, the Java Native Interface and internal non API methods of Java can be used to read and write from native memory.

In general, there are two ways to achieve this goal: One option is to use the Java implementation of Python: Jython. Native CPython extensions can be accessed through the Jython Native Interface. The Jython version, however, is outdated and only Jython 2.7 is available ([Python 3 statement](https://python3statement.org)!!) and Jython will always lag behind CPython releases.

The other approach is to start Java within a CPython process and this is the path that I followed.

This involves the following stepts:

First, a JVM needs to be launched from within a CPython process. A compatibility layer must expose Java classes and methods to the Python interpreter and make them accessible to the user. I used the existing PyJNIus library of the kivy framework for this compatibility layer.

Second, ImgLib2 and NumPy need to be able to understand each other's data. For that purpose I created
 - *imglib2-unsafe* to read from and write into native memory through ImgLib2
 - *imglib2-imglyb* ImgLib2 data structures and wrappers that understand NumPy meta data such as shape, strides, or offsets into native memory
 - *imglyb* the Python interface that easily translates numpy arrays into ImgLib2 data structures with shared memory access

This native memory access is only possible because of the virtualization of pixel access in ImgLib2: algorithms processing ImgLib2 data structures are generally not aware if the data is provided as Java array or native array.

Going back to the initial metaphor of two peaks that are separated by a deep chasm: With imglyb (with a y), I built a bridge over that chasm to connect the SciPy and ImageJ peaks and make interoperability easier and more efficient through shared memory.

I hope that you all are psyched now and ready to install imglyb because it is really easy to get: The most simple way is to install imglyb from conda. I made imglyb and its PyJNIus dependency available on conda-forge to make sure that it will be available and up to date in the future.

Examples are available on PyPI and I created notebooks for more interactive demonstration. I will show some of those notebooks in a few minutes.

How can we use imglyb? It is as simple as `import imglyb`. It is important to note that `imglyb` must be imported before any `jnius` import with the exception of `jnius_config`.

For example, these lines will wrap and display 3D `numpy.ndarray`s: First, import `imglyb` and `numpy`, then create some random data with numpy, wrap it, and display it with BigDataViewer. A second uint32 image is wrapped as ARGB image and displayed in BigDataViewer as well.

More complex examples are available in the `imglyb-examples` package:

In the following I will go through four short example notebooks to demonstrate how to use `imglyb`:

Example 1: I will open a HDF5 file with the `h5` package and display the NumPy array in BigDataViewer. BigDataViewer is a Java viewer for arbitrarily large multi-view 3D images and 3D image sequences and developed by Tobias Pietzsch. Show Example!

Example 2: I will implement an extension for BigDataViewer with which I can paint into a 3D NumPy array.

Example 3: BigWarp is a tool for interactive alignment of 2D or 3D images through pairs of manually placed landmarks. This is used, for example, to align micrographs of different modalities, light and EM, of different specimen of the same animal. I will demonstrate how to warp a 2D NumPy array with BigWarp.

Example 4: Paintera is a tool for the efficient generation of dense ground truth annotations nad proof-reading 3D EM connectomics with 3D visualization and mesh generation on the fly. I will demonstrate how to use Paintera to visualize NumPy label data without the need to pre-compute triangle meshes.

This brings me to the last few slides of my talk. Currently, graphical user interfaces built upon the Java awt or Swing frameworks, e.g. BigDataViewer, require a wrapper script on OSX. This wrapper script is installed with imglyb. Dask arrays cannot be wrapped into ImgLib2 cell images.

In my metaphor I used these logos. I link the organizations or the source on this page.

Finally, I would like to thank you all for attending this talk and I hope that you all are excited that you can share memory between SciPy and ImageJ/ImgLib2 now! As a reminder, I placed the installation instructions and important links on this last slide. The slides are available online at the last link. I will leave Saturday afternoon but should be around for the sprints on Saturday morning to help setup `imglyb` or to give advice for hacking `imglyb` if people are interested.

Thank you for your attention

