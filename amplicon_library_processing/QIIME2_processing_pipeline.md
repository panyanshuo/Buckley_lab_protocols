{
 cells: [
  {
   cell_type: markdown,
   metadata: {},
   source: [
    > ### Pipeling to Process Raw Sequences into Phyloseq Object with DADA2 ###\n,
    * Prep for Import to QIIME2  (Combine two index files)\n,
    * Import to QIIME2\n,
    * Demultiplex\n,
    * Denoise and Merge\n,
    * Prepare OTU Tables and Rep Sequences  *(Note: sample names starting with a digit will break this step)*\n,
    * Classify Seqs\n,
    \n,
    *100% Appropriated from the \Atacama Desert Tutorial\ for QIIME2*\n,
    \n,
    ### Pipeline can handle both 16S rRNA gene and ITS sequences (in theory)####\n,
    * Tested on 515f and 806r\n,
    * Tested on ITS1\n,
    \n,
    ### Commands to Install Dependencies ####\n,
    ##### || QIIME2 ||\n,
    *  conda create -n qiime2-pipeline --file https://data.qiime2.org/distro/core/qiime2-2017.11-conda-linux-64.txt\n,
    * source activate qiime2-pipeline\n,
    \n,
    ** Note: QIIME2 is still actively in development, and I've noticed frequent new releases. Check for the most up-to-date conda install file <https://docs.qiime2.org/2017.11/install/native/#install-qiime-2-within-a-conda-environment>\n,
    \n,
    \n,
    ##### || Copyrighter rrn Database ||\n,
    * The script will automatically install the curated GreenGenes rrn attribute database\n,
    * https://github.com/fangly/AmpliCopyrighter\n,
    \n,
    ##### || rpy2 (don't use conda version) ||\n,
    * pip install rpy2  \n,
    \n,
    ##### || phyloseq ||\n,
    * conda install -c r r-igraph \n,
    * Rscript -e \source('http://bioconductor.org/biocLite.R');biocLite('phyloseq')\ \n,
    \n,
    ##### || R packages ||\n,
    * ape   (natively installed with in conda environment)\n,
    \n,
    \n,
    ### Citations ###\n,
    * Caporaso, J. G., Kuczynski, J., Stombaugh, J., Bittinger, K., Bushman, F. D., Costello, E. K., *et al.* (2010). QIIME allows analysis of high-throughput community sequencing data. Nature methods, 7(5), 335-336.\n,
    \n,
    \n,
    * McMurdie and Holmes (2013) phyloseq: An R Package for Reproducible Interactive Analysis and Graphics of Microbiome Census Data. PLoS ONE. 8(4):e61217\n,
    \n,
    \n,
    * Paradis E., Claude J. & Strimmer K. 2004. APE: analyses of phylogenetics and evolution in R language. Bioinformatics 20: 289-290.\n,
    \n,
    \n,
    * Angly, F. E., Dennis, P. G., Skarshewski, A., Vanwonterghem, I., Hugenholtz, P., & Tyson, G. W. (2014). CopyRighter: a rapid tool for improving the accuracy of microbial community profiles through lineage-specific gene copy number correction. Microbiome, 2(1), 11.\n,
    \n,
    ###### Last Modified by R. Wilhelm on October 12th, 2017 ######\n
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 1: User Input
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    import os, re\n,
    \n,
    # Provide the directory for your index and read files (you can do multiple independently in one go)\n,
    bioblitz = '/home/roli/BioBlitz.2017/SV_based/'\n,
    \n,
    # Prepare an object with the name of the library, the name of the directory object (created above), and the metadatafile name\n,
    #datasets = [['name',directory1,'metadata1','domain of life'],['name',directory2,'metadata2','domain of life']]\n,
    datasets = [['bioblitz',bioblitz,'metadata.tsv','bacteria']]\n,
    \n,
    # Ensure your reads files are named accordingly (or modify to suit your needs)\n,
    readFile1 = 'read1.fq.gz'\n,
    readFile2 = 'read2.fq.gz'\n,
    indexFile1 = 'index_read1.fq.gz'\n,
    indexFile2 = 'index_read2.fq.gz'\n,
    \n,
    ## Enter Minimum Support for Keeping QIIME Classification\n,
    # Note: Classifications that do not meet this criteria will simply be retained, but labeled 'putative'\n,
    min_support = 0.8
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 2: Concatenate Barcodes for QIIME2 Pipeline
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    ## Note: QIIME takes a single barcode file. The command 'extract_barcodes.py' concatenates the forward and reverse read barcode and attributes it to a single read.\n,
    \n,
    # See http://qiime.org/tutorials/processing_illumina_data.html\n,
    \n,
    for dataset in datasets:\n,
        directory = dataset[1]\n,
        index1 = directory+indexFile1\n,
        index2 = directory+indexFile2\n,
        \n,
        # Run extract_barcodes to merge the two index files\n,
        !python2 /opt/anaconda2/bin/extract_barcodes.py --input_type barcode_paired_end -f $index1 -r $index2 --bc1_len 8 --bc2_len 8 -o $directory/output\n,
    \n,
        # QIIME2 import requires a directory containing files names: forward.fastq.gz, reverse.fastq.gz and barcodes.fastq.gz \n,
        !ln -s $directory$readFile1 $directory/output/forward.fastq.gz\n,
        !ln -s $directory$readFile2 $directory/output/reverse.fastq.gz\n,
        \n,
        # Gzip the barcodes files (apparently necessary)\n,
        !pigz -p 5 $directory/output/barcodes.fastq\n,
    \n,
        # Removed orphaned reads files (not needed)\n,
        !rm $directory/output/reads?.fastq\n
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 3: Import into QIIME2
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        \n,
        os.system(' '.join([\n,
            \qiime tools import\,\n,
            \--type EMPPairedEndSequences\,\n,
            \--input-path \+directory+\output/\,\n,
            \--output-path \+directory+\output/\+name+\.qza\\n,
        ]))\n,
        \n,
        # This more direct command is broken by the fact QIIME uses multiple dashes in their arguments (is my theory)\n,
        #!qiime tools import --type EMPPairedEndSequences --input-path $directory/output --output-path $directory/output/$name.qza\n,
         
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 4: Demultiplex
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    ########\n,
    ## Note: The barcode you supply to QIIME is now a concatenation of your forward and reverse barcode.\n,
    # Your 'forward' barcode is actually the reverse complement of your reverse barcode and the 'reverse' is your forward barcode. The file 'primers.complete.csv' provides this information corresponding to the Buckley Lab 'primer number'\n,
    # This quirk could be corrected in how different sequencing facilities pre-process the output from the sequencer\n,
    \n,
    ##\n,
    ## SLOW STEP (~ 2 - 4 hrs)\n,
    ##\n,
    \n,
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        metadata = dataset[2]\n,
        \n,
        os.system(' '.join([\n,
            \qiime demux emp-paired\,\n,
            \--m-barcodes-file \+directory+metadata,\n,
            \--m-barcodes-category BarcodeSequence\,\n,
            \--i-seqs \+directory+\output/\+name+\.qza\,\n,
            \--o-per-sample-sequences \+directory+\output/\+name+\.demux\\n,
        ]))\n,
        \n,
        # This more direct command is broken by the fact QIIME uses multiple dashes in their arguments (is my theory)\n,
        #!qiime demux emp-paired --m-barcodes-file $directory/$metadata --m-barcodes-category BarcodeSequence --i-seqs $directory/output/$name.qza --o-per-sample-sequences $directory/output/$name.demux\n,
        
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 5: Visualize Quality Scores and Determine Trimming Parameters
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {},
   outputs: [],
   source: [
    ## Based on the Graph Produced using the Following Command enter the trim and truncate values. Trim refers to the start of a sequence and truncate the total length (i.e. number of bases to remove from end)\n,
    \n,
    # The example in the Atacam Desert Tutorial trims 13 bp from the start of each read and does not remove any bases from the end of the 150 bp reads:\n,
    #  --p-trim-left-f 13 \\  \n,
    #  --p-trim-left-r 13 \\\n,
    #  --p-trunc-len-f 150 \\\n,
    #  --p-trunc-len-r 150\n,
    \n,
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        \n,
        os.system(' '.join([\n,
            \qiime demux summarize\,\n,
            \--i-data \+directory+\/output/\+name+\.demux.qza\,\n,
            \--o-visualization \+directory+\/output/\+name+\.demux.QC.summary.qzv\\n,
        ]))\n,
        \n,
        ## Take the output from this command and drop it into:\n,
        #https://view.qiime2.org\n,
    \n,
    wait_for_user = input(\The script will now wait for you to input trimming parameters in the next cell. You will need to take the .qzv files for each library and visualize them at <https://view.qiime2.org>. This is hopefully temporary, while QIIME2 developers improve on q2view.\\n\\n[ENTER ANYTHING. THIS IS ONLY MEANT TO PAUSE THE PIPELING]\)\n,
    print(\\\nThe script is now proceeding. Stay tuned to make sure trimming works.\)
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 6: Trimming Parameters | USER INPUT REQUIRED
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    ## User Input Required\n,
    trim_dict = {}\n,
    \n,
    ## Input your trimming parameters into a python dictionary for all libraries\n,
    #trim_dict[\LibraryName1\] = [trim_forward, truncate_forward, trim_reverse, truncate_reverse]\n,
    #trim_dict[\LibraryName2\] = [trim_forward, truncate_forward, trim_reverse, truncate_reverse]\n,
    \n,
    ## Example\n,
    trim_dict[\bioblitz\] = [1, 240, 1, 190]
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 7: Trim, Denoise and Join (aka 'Merge') Reads Using DADA2
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    ## Hack for Multithreading\n,
    # I hardcoded 'nthreads' in both versions of 'run_dada_paired.R' (find your versions by running 'locate run_dada_paired.R' from your home directory)\n,
    # I used ~ 20 threads and the processing finished in ~ 7 - 8hrs\n,
    \n,
    ##\n,
    ## SLOW STEP (~ 6 - 8 hrs, IF multithreading is used)\n,
    ##\n,
    \n,
    \n,
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        \n,
        os.system(' '.join([\n,
            \qiime dada2 denoise-paired\,\n,
            \--i-demultiplexed-seqs \+directory+\/output/\+name+\.demux.qza\,\n,
            \--o-table \+directory+\/output/\+name+\.table\,\n,
            \--o-representative-sequences \+directory+\/output/\+name+\.rep.seqs.final\,\n,
            \--p-trim-left-f \+str(trim_dict[name][0]),\n,
            \--p-trim-left-r \+str(trim_dict[name][2]),\n,
            \--p-trunc-len-f \+str(trim_dict[name][1]),\n,
            \--p-trunc-len-r \+str(trim_dict[name][3])\n,
        ]))\n,
      \n
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 8: Create Summary of OTUs
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        metadata = dataset[2]\n,
        \n,
        os.system(' '.join([\n,
            \qiime feature-table summarize\,\n,
            \--i-table \+directory+\/output/\+name+\.table.qza\,\n,
            \--o-visualization \+directory+\/output/\+name+\.table.qzv\,\n,
            \--m-sample-metadata-file \+directory+metadata\n,
        ]))\n,
    \n,
        os.system(' '.join([\n,
            \qiime feature-table tabulate-seqs\,\n,
            \--i-data \+directory+\/output/\+name+\.rep.seqs.final.qza\,\n,
            \--o-visualization \+directory+\/output/\+name+\.rep.seqs.final.qzv\\n,
        ])) 
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 9: Make Phylogenetic Tree
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    ## Hack for Multithreading\n,
    # I hardcoded 'n_threads' in '_mafft.py' in the directory ~/anaconda3/envs/qiime2-2017.9/lib/python3.5/site-packages/q2_alignment\n,
    # I used ~ 20 threads and the processing finished in ~ 15 min\n,
    \n,
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        metadata = dataset[2]\n,
        domain = dataset[3]\n,
    \n,
        if domain != \fungi\:\n,
            # Generate Alignment with MAFFT\n,
            os.system(' '.join([\n,
                \qiime alignment mafft\,\n,
                \--i-sequences \+directory+\/output/\+name+\.rep.seqs.final.qza\,\n,
                \--o-alignment \+directory+\/output/\+name+\.rep.seqs.aligned.qza\\n,
            ]))\n,
    \n,
            # Mask Hypervariable parts of Alignment\n,
            os.system(' '.join([\n,
                \qiime alignment mask\,\n,
                \--i-alignment \+directory+\/output/\+name+\.rep.seqs.aligned.qza\,\n,
                \--o-masked-alignment \+directory+\/output/\+name+\.rep.seqs.aligned.masked.qza\\n,
            ])) \n,
    \n,
            # Generate Tree with FastTree\n,
            os.system(' '.join([\n,
                \qiime phylogeny fasttree\,\n,
                \--i-alignment \+directory+\/output/\+name+\.rep.seqs.aligned.masked.qza\,\n,
                \--o-tree \+directory+\/output/\+name+\.rep.seqs.tree.unrooted.qza\\n,
            ])) \n,
    \n,
            # Root Tree\n,
            os.system(' '.join([\n,
                \qiime phylogeny midpoint-root\,\n,
                \--i-tree \+directory+\/output/\+name+\.rep.seqs.tree.unrooted.qza\,\n,
                \--o-rooted-tree \+directory+\/output/\+name+\.rep.seqs.tree.final.qza\\n,
            ])) \n
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 10: Classify Seqs
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        metadata = dataset[2]\n,
        domain = dataset[3]\n,
    \n,
        # Classify\n,
        if domain == 'bacteria':\n,
            os.system(' '.join([\n,
                \qiime feature-classifier classify-sklearn\,\n,
                \--i-classifier /home/db/GreenGenes/qiime2_13.8.99_515.806_nb.classifier.qza\,\n,
                \--i-reads \+directory+\/output/\+name+\.rep.seqs.final.qza\,\n,
                \--o-classification \+directory+\/output/\+name+\.taxonomy.final.qza\\n,
            ]))\n,
    \n,
        if domain == 'fungi':\n,
            os.system(' '.join([\n,
                \qiime feature-classifier classify-sklearn\,\n,
                \--i-classifier /home/db/UNITE/qiime2_unite_ver7.99_20.11.2016_classifier.qza\,\n,
                \--i-reads \+directory+\/output/\+name+\.rep.seqs.final.qza\,\n,
                \--o-classification \+directory+\/output/\+name+\.taxonomy.final.qza\\n,
            ]))\n,
    \n,
        # Output Summary\n,
        os.system(' '.join([\n,
            \qiime metadata tabulate\,\n,
            \--m-input-file \+directory+\/output/\+name+\.taxonomy.final.qza\,\n,
            \--o-visualization \+directory+\/output/\+name+\.taxonomy.final.summary.qzv\\n,
        ])) 
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 11: Prepare Data for Import to Phyloseq
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    ## Make Function to Re-Format Taxonomy File to Contain Full Column Information \n,
    # and factor in the certain of the taxonomic assignment\n,
    \n,
    def format_taxonomy(tax_file, min_support):\n,
        output = open(re.sub(\.tsv\,\.fixed.tsv\,tax_file), \w\)\n,
        output.write(\\\t\.join([\OTU\,\Domain\,\Phylum\,\Class\,\Order\,\Family\,\Genus\,\Species\])+\\\n\)\n,
        \n,
        with open(tax_file, \r\) as f:\n,
            next(f) #skip header\n,
    \n,
            for line in f:\n,
                line = line.strip()\n,
                line = line.split(\\\t\)\n,
    \n,
                read_id = line[0]\n,
                tax_string = line[1]\n,
    \n,
                # Annotate those strings which do not meet minimum support\n,
                if float(line[2]) < float(min_support):\n,
                    tax_string = re.sub(\__\,\__putative \,tax_string)\n,
    \n,
                # Remove All Underscore Garbage (gimmie aesthetics)\n,
                tax_string = re.sub(\k__|p__|c__|o__|f__|g__|s__\,\\,tax_string) \n,
    \n,
                # Add in columns containing unclassified taxonomic information\n,
                # Predicated on maximum 7 ranks (Domain -> Species)\n,
                full_rank = tax_string.split(\;\)\n,
                last_classified = full_rank[len(full_rank)-1]\n,
    \n,
                count = 1\n,
                while last_classified == \ \:\n,
                    last_classified = full_rank[len(full_rank)-count]\n,
                    count = count + 1\n,
    \n,
    \n,
                for n in range(full_rank.index(last_classified)+1, 7, 1):\n,
                    try:\n,
                        full_rank[n] = \unclassifed \+last_classified\n,
                    except:\n,
                        full_rank.append(\unclassifed \+last_classified)\n,
    \n,
                output.write(read_id+\\\t\+'\\t'.join(full_rank)+\\\n\)\n,
                \n,
        return()
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {
    collapsed: true
   },
   outputs: [],
   source: [
    #####################\n,
    ## Export from QIIME2\n,
    \n,
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        metadata = dataset[2]\n,
        domain = dataset[3]\n,
    \n,
        ## Final Output Names\n,
        fasta_file = directory+\/output/\+name+\.rep.seqs.final.fasta\\n,
        tree_file = directory+\/output/\+name+\.tree.final.nwk\\n,
        tax_file = directory+\/output/\+name+\.taxonomy.final.tsv\\n,
        count_table = directory+\/output/\+name+\.counts.final.biom\\n,
    \n,
        # Export Classifications\n,
        os.system(' '.join([\n,
            \qiime tools export\,\n,
            directory+\/output/\+name+\.taxonomy.final.qza\,\n,
            \--output-dir \+directory+\/output/\\n,
        ]))\n,
        \n,
        # Reformat Classifications to meet phyloseq format\n,
        format_taxonomy(directory+\/output/taxonomy.tsv\, min_support)\n,
    \n,
        # Export SV Table\n,
        os.system(' '.join([\n,
            \qiime tools export\,\n,
            directory+\/output/\+name+\.table.qza\,\n,
            \--output-dir \+directory+\/output/\\n,
        ]))\n,
    \n,
        # Export SV Sequences\n,
        os.system(' '.join([\n,
            \qiime tools export\,\n,
            directory+\/output/\+name+\.rep.seqs.final.qza\,\n,
            \--output-dir \+directory+\/output/\\n,
        ]))\n,
        \n,
        # Export Tree\n,
        os.system(' '.join([\n,
            \qiime tools export\,\n,
            directory+\/output/\+name+\.rep.seqs.tree.final.qza\,\n,
            \--output-dir \+directory+\/output/\\n,
        ]))\n,
        \n,
        # Rename Exported Files\n,
        %mv $directory/output/dna-sequences.fasta $fasta_file\n,
        %mv $directory/output/feature-table.biom $count_table\n,
        %mv $directory/output/taxonomy.fixed.tsv $tax_file\n,
        \n,
        if domain == \bacteria\:\n,
            %mv $directory/output/tree.nwk $tree_file\n,
        \n
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 13: Get 16S rRNA Gene Copy Number (rrn)
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {},
   outputs: [],
   source: [
    ## This step is based on the database contructed for the software 'copyrighter'\n,
    ## The software itself lacked information about datastructure (and, the import of a biom from QIIME2 failed, likely because there are multiple versions of the biom format)\n,
    downloaded = \N\\n,
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        domain = dataset[3]\n,
    \n,
        if domain == 'bacteria':\n,
            if downloaded == \N\:\n,
                ## Download copyrighter database\n,
                !git clone https://github.com/fangly/AmpliCopyrighter $directory/temp/\n,
    \n,
                ## There are multiple GreenGenes ID numbers for a given taxonomic string.\n,
                ## However, the copyrighter database uses the same average rrn copy number.\n,
                ## We will therefore just use the taxonomic strings, since QIIME2 does not output the ID numbers\n,
    \n,
                !sed -e '1,1075178d; 1078115d' $directory/temp/data/201210/ssu_img40_gg201210.txt > $directory/output/copyrighter.tax.strings.tsv\n,
    \n,
                ## Create Dictionary of rrnDB\n,
                rrnDB = {}\n,
    \n,
                with open(directory+\/output/copyrighter.tax.strings.tsv\, \r\) as f:\n,
                    for line in f:\n,
                        line = line.strip()\n,
                        line = line.split(\\\t\)\n,
    \n,
                        try:\n,
                            rrnDB[line[0]] = line[1]\n,
    \n,
                        except:\n,
                            pass\n,
    \n,
                downloaded = \Y\\n,
                \n,
            ## Attribute rrn to readID from taxonomy.tsv\n,
            output = open(directory+\/output/\+name+\.seqID.to.rrn.final.tsv\,\w\)\n,
            output.write(\Feature ID\\trrn\\n\)\n,
    \n,
            with open(directory+\/output/taxonomy.tsv\, \r\) as f:\n,
                missing = 0\n,
                total = 0\n,
                next(f)  # Skip Header\n,
    \n,
                for line in f:\n,
                    line = line.strip()\n,
                    line = line.split(\\\t\)\n,
    \n,
                    seqID = line[0]\n,
    \n,
                    try:\n,
                        rrn = rrnDB[line[1]]\n,
    \n,
                    except:\n,
                        rrn = \NA\\n,
                        missing = missing + 1\n,
    \n,
                    total = total + 1\n,
                    output.write(seqID+\\\t\+rrn+\\\n\)\n,
    \n,
            print(\\\nPercent of OTUs Missing {:.1%}\.format(float(missing)/total))\n,
            print(\Don't Panic! The majority of missing OTUs could be low abundance.\)
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 14: Import into Phyloseq
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {},
   outputs: [],
   source: [
    ## Setup R-Magic for Jupyter Notebooks\n,
    import rpy2\n,
    %load_ext rpy2.ipython\n,
    \n,
    def fix_biom_conversion(file):\n,
        with open(file, 'r') as fin:\n,
            data = fin.read().splitlines(True)\n,
        with open(file, 'w') as fout:\n,
            fout.writelines(data[1:])
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {},
   outputs: [],
   source: [
    import pandas as pd\n,
    %R library(phyloseq)\n,
    %R library(ape)\n,
    \n,
    \n,
    for dataset in datasets:\n,
        name = dataset[0]\n,
        directory = dataset[1]\n,
        metadata = dataset[2]\n,
        domain = dataset[3]\n,
     \n,
        #### IMPORT DATA to R\n,
        ## For '.tsv' files, use Pandas to create a dataframe and then pipe that to R\n,
        ## For '.biom' files, first convert using 'biom convert' on the command-line\n,
        ## Had problems importing the count table with pandas, opted for using read.table in R\n,
        \n,
        # Import Taxonomy File\n,
        tax_file = pd.read_csv(directory+\/output/\+name+\.taxonomy.final.tsv\, sep=\\\t\)\n,
        %R -i tax_file\n,
        %R rownames(tax_file) = tax_file$OTU\n,
        %R tax_file$OTU <- NULL\n,
        %R tax_file <- tax_file[sort(row.names(tax_file)),] #read names must match the count_table\n,
    \n,
        # Import Sample Data\n,
        #sample_file = pd.read_csv(directory+\/\+metadata, sep=\\\t\)\n,
        sample_file = pd.read_table(directory+metadata, keep_default_na=False)\n,
        %R -i sample_file\n,
        %R rownames(sample_file) = sample_file$X.SampleID   \n,
        %R sample_file$X.SampleID <- NULL\n,
        %R sample_file$LinkerPrimerSequence <- NULL  ## Clean-up some other stuff\n,
        \n,
        # Import Count Data\n,
        os.system(' '.join([\n,
            \biom convert\,\n,
            \-i\,\n,
            directory+\/output/\+name+\.counts.final.biom\,\n,
            \-o\,\n,
            directory+\/output/\+name+\.counts.final.tsv\,\n,
            \--to-tsv\\n,
        ]))\n,
        \n,
        # The biom converter adds a stupid line that messes with the table formatting\n,
        fix_biom_conversion(directory+\/output/\+name+\.counts.final.tsv\)\n,
    \n,
        # Finally import\n,
        count_table = pd.read_csv(directory+\/output/\+name+\.counts.final.tsv\, sep=\\\t\)\n,
        %R -i count_table\n,
        %R rownames(count_table) = count_table$X.OTU.ID   \n,
        %R count_table$X.OTU.ID <- NULL    \n,
        %R count_table <- count_table[sort(row.names(count_table)),] #read names must match the tax_table\n,
        \n,
        # Convert to Phyloseq Objects\n,
        %R p_counts = otu_table(count_table, taxa_are_rows = TRUE)    \n,
        %R p_samples = sample_data(sample_file)    \n,
        %R p_tax = tax_table(tax_file)\n,
        %R taxa_names(p_tax) <- rownames(tax_file) # phyloseq throws out rownames\n,
        %R colnames(p_tax) <- colnames(tax_file) # phyloseq throws out colnames\n,
        \n,
        # Merge Phyloseq Objects\n,
        %R p = phyloseq(p_counts, p_tax)\n,
    \n,
        # Import Phylogenetic Tree\n,
        if domain == \bacteria\:\n,
            tree_file = directory+\/output/\+name+\.tree.final.nwk\\n,
            %R -i tree_file  \n,
            %R p_tree <- read.tree(tree_file)\n,
        \n,
            # Combine All Objects into One Phyloseq\n,
            %R p_final <- merge_phyloseq(p, p_samples, p_tree)\n,
        \n,
        else:\n,
            # Combine All Objects into One Phyloseq\n,
            %R p_final <- merge_phyloseq(p, p_samples)\n,
            \n,
        # Save Phyloseq Object as '.rds'\n,
        output = directory+\/output/p_\+name+\.final.rds\\n,
        %R -i output\n,
        %R saveRDS(p_final, file = output)\n,
        \n,
        # Confirm Output\n,
        %R print(p_final)
   ]
  },
  {
   cell_type: markdown,
   metadata: {},
   source: [
    # Step 15: Clean-up Intermediate Files and Final Outputs
   ]
  },
  {
   cell_type: code,
   execution_count: null,
   metadata: {},
   outputs: [],
   source: [
    for dataset in datasets:\n,
        directory = dataset[1]\n,
        metadata = dataset[2]\n,
        \n,
        # Remove Files\n,
        if domain == \bacteria\:\n,
            %rm -r $directory/output/*tree.unrooted.qza \n,
            %rm -r $directory/output/*aligned.masked.qza \n,
            \n,
        %rm $directory/output/*.biom \n,
        %rm -r $directory/temp/\n,
        %rm $directory/output/*barcodes.fastq.gz \n,
        %rm $directory/output/taxonomy.tsv\n,
        %rm $directory/output/forward.fastq.gz # Just the symlink\n,
        %rm $directory/output/reverse.fastq.gz # Just the symlink\n,
        %rm $directory/output/copyrighter.tax.strings.tsv\n,
        \n,
        # Separate Final Files\n,
        %mkdir $directory/final/    \n,
        %mv $directory/output/*.final.rds $directory/final/\n,
        %mv $directory/output/*.taxonomy.final.tsv $directory/final/    \n,
        %mv $directory/output/*.counts.final.tsv $directory/final/\n,
        %mv $directory/output/*.final.fasta $directory/final/\n,
        %cp $directory$metadata $directory/final/\n,
        %mv $directory/output/*.seqID.to.rrn.final.tsv $directory/final/ \n,
        %mv $directory/output/*.nwk $directory/final/ \n,
        \n,
        # Gzip and Move Intermediate Files\n,
        !pigz -p 10 $directory/output/*.qza\n,
        !pigz -p 10 $directory/output/*.qzv\n,
        \n,
        %mv $directory/output/ $directory/intermediate_files\n,
    \n,
    print(\Your sequences have been successfully saved to 'final' and 'intermediate_files'\)
   ]
  }
 ],
 metadata: {
  anaconda-cloud: {},
  hide_input: true,
  kernelspec: {
   display_name: Environment (conda_qiime2-pipeline),
   language: python,
   name: conda_qiime2-pipeline
  },
  language_info: {
   codemirror_mode: {
    name: ipython,
    version: 3
   },
   file_extension: .py,
   mimetype: text/x-python,
   name: python,
   nbconvert_exporter: python,
   pygments_lexer: ipython3,
   version: 3.5.4
  }
 },
 nbformat: 4,
 nbformat_minor: 1
}