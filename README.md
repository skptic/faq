# Nextflow FAQ

#### 1. I have a collection of input files (e.g. `carrots.fa`, `onions.fa`, `broccoli.fa`). How can I specify a process to be performed on each file in a parallel manner?

The idea here is to create a `channel` that will trigger a process execution for each of your files. 

First define a parameter that specifies where the input files are:

    params.input = "data/*.fa"

Each of the files in the data directory can be made into a channel with:

    vegetable_datasets = Channel.fromPath(params.input)

From here, each time the variable `vegetable_datasets` is called as an input to a process, the process will be performed
on each of the files in the vegetable datasets. For example, each input file may contain a collection of unaligned sequences.
We can specify a process to align them as follows:

    process clustalw2_align {
        input:
        file vegetable_fasta from vegetable_datasets

        output:
        file "${vegetable_fasta.baseName}.aln" into vegetable_alns

        script:
        """
        clustalw2 -INFILE=${vegetable_fasta}
        """
    }

This would result in the alignment of the three vegetable fasta files into `carrots.aln`, `onions.aln` and `broccoli.aln`.

These aligned files are now in the channel `vegetable_alns` and can be used as input for a further process.

#### 2. How do I get a unique identifier based on a dataset file names (e.g. `broccoli` from `broccoli.fa`) and have the results going to a specific folder (e.g. `results/broccoli/`)?

First we can specify a results directory as shown below:

    results_path = $PWD/results

The best way to manage this is to have the channel emit a tuple containing both the file base name (`broccoli`) and the full file path (`data/broccoli.fa`):

    datasets = Channel
                    .fromPath(params.input)
                    .map { file -> tuple(file.baseName, file) }

And in the process we can then reference these variables (`datasetID` and `datasetFile`):

    process clustalw2_align {
        publishDir "$results_path/$datasetID"

        input:
        set datasetID, file(datasetFile) from datasets

        output:
        set datasetID, file("${datasetID}.aln") into aligned_files

        script:
        """
        clustalw2 -INFILE=${datasetFile} -OUTFILE=${datasetID}.aln
        """
    }


In our example above would now have the folder `broccoli` which would contain the file `broccoli.aln`.

Channels can contain and emit any type of data structure simplifying the flow of data.

#### 3. Can a channel be used in two input statements? For example, I want `carrots.fa` to be aligned by both *ClustalW* and *T-Coffee*.

No, a channel can be consumed only by one process or operator. You must split a channel before calling it as an input in 
different processes. First we create the channel emitting the input files:

    vegetable_datasets = Channel.fromPath(params.input)

Next we can split it into two channels:

    vegetable_datasets.into { datasets_clustalw; datasets_tcoffee }


Then we can define a process for aligning the datasets with *ClustalW*:

    process clustalw2_align {
        input:
        file vegetable_fasta from datasets_clustalw
        
        output:
        file "${vegetable_fasta.baseName}.aln" into clustalw_alns

        script:
        """
        clustalw2 -INFILE=${vegetable_fasta}
        """
    }


And a process for aligning the datasets with *T-Coffee*:

    process tcoffee_align {
        input:
        file vegetable_fasta from datasets_tcoffee
        
        output:
        file "${vegetable_fasta.baseName}.aln" into tcoffee_alns
    
        script:
        """
        t_coffee ${vegetable_fasta}
        """
    }

The upside of splitting the channels is that given our three unaligned fasta files (`broccoli.fa`, `onion.fa` and `carrots.fa`) 
six alignment processes (three x ClustalW) + (three x T-Coffee) will be executed as parallel processes.

#### 4. Why the hell do I keep getting “did you forget to escape \${5} errors”? Variables explained.
    
How to I use a Nextflow variable and in the script part of process?
How to I use a Bash/Perl/Python variable and in the script part of process?
How to I make a Bash/Perl/Python variable a Nextflow variable.

#### 5. I have executables in my code, how should I call them in Nextflow?

Nextflow will automatically add the directory `bin` into the `PATH` environmental variable. So therefore any
executable in the `bin` folder of a Nextflow pipeline can be called without the need to reference the full path.

For example, we may wish to reformat our *ClustalW* alignments from Question 3 into *PHYLIP* format. We will use the
handy tool `esl-reformat` for this task.

First we place copy (or create a symlink to) the `esl-reformat` executable to the project's bin folder. From above we see the
*ClustalW* alignments are in the channel `clustalw_alns`:

    process phylip_reformat {
        input:
        file clustalw_alignment from clustalw_alns
        
        output:
        file "${clustalw_alignment.baseName}.phy" to clustalw_phylips

        script:
        """
        esl-reformat phylip ${clustalw_alignment} ${clustalw_alignment.baseName}.phy
        """
    }


    process generate_bootstrap_replicates {
        input:
        file clustalw_phylip from clustalw_phylips
        output:
        file "${clustalw_alignment.baseName}.phy" to clustalw_phylips

        script:
        """
        esl-reformat phylip ${clustalw_alignment} ${clustalw_alignment.baseName}.phy
        """
    }


#### 6. How do I iterate over a process i times?

To perform a process *i* times, we can specify the input to be `each x from y..z`. For example::

    bootstrapReplicates=100

    process bootstrapReplicateTrees {
        publishDir "$results_path/$datasetID/bootstrapsReplicateTrees"

        input:
        each x from 1..bootstrapReplicates
        set val(datasetID), file(ClustalwPhylips)

        output:
        file "bootstrapTree_${x}.nwk" into bootstrapReplicateTrees

        script:
        // Generate Bootstrap Trees

        """
        raxmlHPC -m PROTGAMMAJTT -n tmpPhylip${x} -s tmpPhylip${x}
        mv "RAxML_bestTree.tmpPhylip${x}" bootstrapTree_${x}.nwk
        """
    }


#### 7. I have *i* files in a channel, how do I iterate over them within a process?

For example, I have 100 files in the `bootstrapReplicateTrees` channel. I wish to perform one process where I iterate
over each file inside the process.

The idea here to transform the channel emitting 100 files to a channel that will collect all files into a list object 
and produce that list as a sole emission. Then the process' script will be able to iterate over the files by using 
a simple for-loop.

    process concatenateBootstrapReplicates {
        publishDir "$results_path/$datasetID/concatenate"

        input:
        file bootstrapTreeList from bootstrapReplicateTrees.toList()

        output:
        file "concatenatedBootstrapTrees.nwk"

        // Concatenate Bootstrap Trees
        script:
        """
        for every treeFile in ${bootstrapTreeList}
        do
            cat \$treeFile >> concatenatedBootstrapTrees.nwk
        done

        """
    }


### 8. I have two channels one with several items and a second one with only one item, why the process is running a single time?

    process intersect_bed {
        input:
        file bed from bed_collection
        file bed_features
        output:
        file ('intersect.bed') into bed_light_activity

        script:
        """
        bedtools intersect -a $bed -b $bed_features > intersect.bed
        """
    }

In this example, the process will be run only one time. Here the channel is a dataflow stream (or queue) and when an item 
is read it is removed from the channel, as the channel contains only one  file,  when it is read, it sends  a termination 
signal to the process and it stops.

If you want to run the process as many times as files in bed_collection, you only need to change the declaration of 
`bed_features` by:

    file bed_features.first()

This code generates a dataflow value (or singleton). This type of channels can be used as many times as needed, because 
it always returns the same value without removing it. That is way with this command the process will be run as many time 
as bed files inside the bed_collection channel.