# mothur-py

Copyright &#169; 2017, Richard Campen. All rights reserved.

See LICENSE.txt for full license conditions.

---

### Description

A python wrapper for the command line version of the bioinformatics tool 
[mothur](https://www.mothur.org/).

Mothur-py was inspired by the [ipython-mothurmagic](https://github.com/SchlossLab/ipython-mothurmagic) module, but with an
intention to provide a more general python wrapper that would work outside of the IPython/Jupyter notebook environment, 
as well as provide support for mothur's `current` keyword functionality.

**Note:** This module has only been tested with mothur v1.39.5 and python 3.6 on Windows 10 (64-bit). It should in 
theory work with other versions of mothur, but the older the version the less likely as this module relies upon some of 
the more recent mothur commands/output to function properly.

---

### Installation

To install the latest release version you can just `pip install mothur-py`. To install the most up to date code you should
download/clone this repository and create a binary distribution using `python setup.py bdist` that will create a zip file
in the `dist` folder. You should then pip install this zip file using `pip install <zip_file_name>`. The advantage of this
method over just running `python setup.py install` is that you can easily remove or update the package via pip.

---

### Basic Usage

The use of this module requires that mothur is in the users `PATH` environment variable.

Use of this module revolves around the `Mothur` class that catches method calls and passes them off to mothur to be run 
as commands. An instance of the `Mothur` class needs to be created before running any commands:

    # create instance of Mothur class
    from mothur_py import Mothur
    m = Mothur()
    
Commands in mothur can then be executed as methods of the `Mothur` class instance using the same names you would use 
within the command line version of mothur:

    # run the mothur help command
    m.help()

Command parameters can either be passed as python native types (i.e. strings, integers, floats, booleans, lists) *or* as
strings that match the format that mothur would expect:

    # running make contigs using str input for file parameter, and int for processor paramenter
    m.make.contigs(file='basic_usage.files', processors=2)

    # running summary.single, passing calculators as mothur formatted list
    m.summary.single(shared='current', calc='nseqs-sobs-coverage-shannon-simpson')

    # running summary.single, passing calculators as python list also works
    m.summary.single(shared='current', calc=['nseqs', 'sobs', 'coverage', 'shannon', 'simpson'])

    
There is also full implementation of the `current` keyword used in the command line version of mothur:
       
    # run the mothur summary.seqs command using the 'current' option
    # NOTE: current is being passed as a string
    m.summary.seqs(fasta='current')
     
    # like the command line version, you don't even need to specify 
    # the 'current' keyword for some commands
    m.summary.seqs() 
    
Behind the scenes, the `current` keyword is enabled by appending the users command with the `get.current()` command to 
list the current directories and files being used by mothur, parsing of the output to extract this information, and 
prepending future commands with `set.dir()` and `set.current()` to tell mothur what these should be. This is necessary 
as each call to mothur is executed as a separate mothur session and therefore mothur can not store this information 
itself.

---

### Configuration 
    
The `Mothur` class stores configuration options for how mothur is executed. These options include `verbosity` to control
how much output there is, `mothur_seed` to control the seed used by mothur for random number generation, and 
`suppress_logfile` which suppresses the creation of the mothur logfile.

When `verbosity` is set to `0` there is no output printed, `1` prints the normal output as would be seen with command 
line execution (minus the header that contains the mothur version and runtime information), and `2` displays all output
including the commands being executed behind the scenes to enable the `current` keyword to work. The default option is 
`0`, with `1` being useful when you want to see the standard mothur output, and `2` being useful for debugging purposes. 

If `mothur_seed` is set to a valid integer then this number will be passed to mothur to be used for random number
generation. This is implemented by adding the `seed=<your seed here>` named parameter to each mothur command. Not all 
commands will accept having a seed set. For these commands you may need to set the `mothur_seed` parameter to `None`
for the execution of that command, e.g.:
 
    m = Mothur(mothur_seed=12345)
    
    # summary.seqs() allows setting the seed so this will run fine
    m.summary.seqs(fasta='current')
    
    # help() does not accept having the seed set so need to alter that value temporarily, otherwise an error will occur
    seed = m.mothur_seed
    m.mothur_seed = None
    m.help()
    m.mothur_seed = seed
    

The `supress_logfile` option is useful when you don't want the log files, such as when running in an Jupyter (nee 
IPython) notebook with `verbosity=1`, in which case you already have a record of mothur's output and the mothur logfiles
are superfluous.

**Note:** Currently, due to the way that mothur creates the logfiles, a logfile will always be created BUT it will be 
cleaned up upon successful execution if `suppress_logfile=True`. However, if mothur fails to successfully execute, i.e. 
execution hangs or is interrupted, the logfile will not be cleaned up. For relevant discussion of this behaviour in 
mothur see [here](https://github.com/mothur/mothur/issues/281) and [here](https://github.com/mothur/mothur/issues/377).

You can also instantiate the `Mothur` object with your desired configuration options.

    m = Mothur(verbosity=1, mothur_seed=543210, suppress_logfile=True)
    
---

### Advanced Usage

The current files and current directories for use in mothur are stored in dictionary attributes of the `Mothur` 
instance, `current_files` and `current_dirs` respectively. These values can be passed to mothur commands, e.g:

    # passing current fasta file to summary.seqs()
    m.summary.seqs(fasta=m.current_files['fasta'])
       
The `current` keyword is actually just a shortcut for this functionality so it will always be easier to just pass 
`'current'`. However, this demonstrates that the parameters of the mothur commands can accept any variable as long as it 
will resolve to something that mothur accepts. In the above example, the dictionary value resolves to a string that is
the path to the `.fasta` file. As a better example of passing python variables as mothur command parameters, you could 
perform classification of sequences at multiple defined cutoffs as follows:

    # iterate over list off possible cutoff values
    for cutoff in [70, 80, 90]:   
        # save outputs to different folders, but keep input the same
        output_dir = 'cutoff_%s' % cutoff
        m.set.dir(output=output_dir, input='.')
        m.classify.seqs(fasta='current', count='current', reference='reference.fasta', taxonomy='referenece.tax',
        cutoff=cutoff)
        
This may be a convoluted example, but it demonstrates the functionality well. One note of caution with this approach is 
that depending on the mothur command and the parameter you are changing, you may be overwriting your output files as you 
go. This is the reason for saving each output to a different folder in the above example.

You can also instantiate a `Mothur` instance with predefined current file and directory dictionaries:

    m = Mothur(current_files=my_predefined_files_dict, current_dirs=my_predefined_files_dict)

This can be convenient for saving and loading the state of a mothur object to/from file as such:

    import json

    # save state of mothur object, m, to json file
    with open('mothur_object.json', 'w') as out_handle:
        json.dump(vars(m), out_handle)

    # can reload mothur object from the json file
    with open('mothur_object.json', 'r') as in_handle:
        m = Mothur(**json.load(in_handle))

You can also modify the contents of these dictionaries in between mothur commands. In the previous example
where we classified at different cutoffs, we could have instead controlled the input and output directories in python
instead of withing mothur:

    for cutoff in [70, 80, 90]:   
        # save outputs to different folders, but keep input the same
        m.current_dirs['output'] = 'cutoff_%s' % cutoff
        m.current_dirs['input'] = '.'
