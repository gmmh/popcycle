Popcycle
========

![travis-ci status](https://travis-ci.org/uwescience/popcycle.svg?branch=master)

The Popcycle pipeline performs 3 different analyses:

1. filtration of Optimally Positioned Particles
2. Manual Gating of cytometric populations
3. Aggregate statistics for the different populations.

The output of each step is saved into a SQL database using `sqlite3`. To run `popcycle` and analyze SeaFlow data in real-time, you need to set the filter and gating parameters, and press play, that's it!

# Installation
1. Download the .zip file of the entire popcycle repository into your computer.

2. Unzip the file

3. Set up the paths of the necessary folders. To do so, edit the file globals.R in path/to/popcyle_repository/R/globals.R. 

4. Open the terminal and go to the directory where popcycle is unzipped, and type:

    ```sh
    $ bash setup.sh
    ```

This creates all the necessary directories, the popcycle database (`popcycle.db`), and installs `popcycle` as an R package. The `setup.R` script should also install `RSQLite` and `splancs` packages if they are not already installed.

WARNINGS: The `setup.sh` has created a popcycle directory in `~/popcycle`. This is different from the popcycle repository. 

# Initialization
1. First step is to set the parameters for the filtration method (notch and width). For this example, we are going to set the gating parameters using the latest evt file collected by the instrument (but you choose any evt file). Open an R session and type:

    ```r
    > library(popcycle)
    > file.name <- get.latest.evt.with.day() # name of the latest evt file collected, with folder name.
    > evt <- readSeaflow(paste(evt.location, file.name, sep='/')) # load the evt file
    > notch <- best.filter.notch(evt, notch=seq(0.1, 1.4, by=0.1),width=0.5, do.plot=TRUE)
    ```
    This function helps you set the best notch (it is not fully tested, so it may not always work...)

    To plot the filtration step, use the following function

    `> plot.filter.cytogram(evt, notch=notch, width=0.5)`
    
  Once you are satisfy with the filter parameters, you can filter `evt` to get `opp` by typing:
  
    `> opp <- filter.notch(evt, notch=notch, width=0.5)`


    IMPORTANT: To save the filter parameters so the filter parmaters will be apply to all new evt files, you need to call the function: 
    
    `> setFilterParams(notch=notch, width=0.5)` 

This function saves the parameters in ~/popcycle/params/filter/filter.csv. Note that every changes in the filter parameters are automatically saved in the logs (~popcycle/logs/filter/filter.csv).


2. Second step is to set the gating for the different populations. WARNINGS: The order in which you gate the different populations is very important, choose it wisely. The gating has to be performed over optimally positioned particles only, not over an evt file. In this example, you are going to first gate the 'beads' (this is always the first population to be gated.). Then we will gate 'Synechococcus' population (this population needs to be gated before you gate Prochlorococcus or picoeukaryote), and finally 'Prochlorococcus' and 'picoeukaryote' population. After drawing your gate on the plot, right-click to finalize.
In the R session, type:

    ```r
    > library(popcycle)
    > file.name <- get.latest.evt() # name of the latest evt file collected
    > opp <- get.opp.by.file(file.name)
    > setGateParams(opp, popname='beads', para.x='fsc_small', para.y='pe')
    > setGateParams(opp, popname='synecho', para.x='fsc_small', para.y='pe')
    > setGateParams(opp, popname='prochloro', para.x='fsc_small', para.y='chl_small')
    > setGateParams(opp, popname='picoeuk', para.x='fsc_small', para.y='chl_small')
    ```

    Similar to the `setFilterParams` function, `setGateParams` saves the gating parameters and order in which the gating was performed in `~/popcycle/params/params.RData`, parameters for each population are also separately saved as a `.csv` file. Note that every changes in the gating parameters are automatically saved in the logs (`~popcycle/logs/params/`).

    Note: If you want to change the order of the gating, delete a population, or simply restartt over, use the function 
    <code>resetGateParams()</code>
    
3. To cluster the different population according to your manual gating, type:

        > vct <- classify.opp(opp, ManualGating)

4. To plot the cytogram with clustered populations, use the following function:

    ```r
    > opp$pop <- vct
    > par(mfrow=c(1,2))
    > plot.vct.cytogram(opp, para.x='fsc_small', para.y='chl_small')
    > plot.vct.cytogram(opp, para.x='fsc_small', para.y='pe')
    ```



# Play

Now that the filter and gating parameters are set, clic PLAY! popcycle will apply the filter and gating parameters for every new evt files collected by the instrument and generate aggregate statistics for each population. All these steps are wrapped into one single function:

`> evaluate.last.evt()`



Here are the 6 steps that the wrapping function is performing. This is for your information but you don't need to execute each of these steps, `evaluate.last.evt()` is doing it for you!

1. First, popcycle loads the filter parameters

    `> params <- read.csv(paste(param.filter.location,"filter.csv", sep='/'))`

2. Filter opp using the `filter.notch` function

    `> opp <- filter.evt(evt, filter.notch, width = params$width, notch = params$notch)`

3. Upload the opp into the database (and saving the cruise ID and the file name for each particle)

    `> upload.opp(opp.to.db.opp(opp, cruise.id, file.name))` where `file.name` represent the filename of the last evt file (using `get.latest.file.with.day()` function).

4. Apply the gating parameters defined by the manual method for each population

    `> vct <- classify.opp(opp, ManualGating)` 

5. Upload the particle assignment into the database (and saving the cruise ID and the file name for each particle).

    `> upload.vct(vct.to.db.vct(vct, cruise.id, file.name, 'Manual Gating'))`

6. Finally, popcycle will performs aggregate statistics for each population. To calculate cell abundance, we need to know the flow rate and acquisition time of the instrument for each file as well as the opp/evt ratio. Informations related to the instrument are automatically recorded in SeaFlow Log files (.sfl) and stored in a table called 'sfl' in the database. The opp/evt ratio is also stored in a table called 'opp_evt_ratio' in the database. Aggregate statistics are then calculated and recorded in a table called 'stats' in the database.

    `> insert.stats.for.file(file.name)`

# Visualization
Data generated for every file can be visualize using a set of functions:

1. To plot the filter steps

    `> plot.filter.cytogram.by.file(file.name)`

2. To plot opp

    `> plot.cytogram.by.file(file.name)`

3. To plot vct

    `> plot.vct.cytogram.by.file(file.name)`

4. To plot aggregate statistics, for instance, cell abundance the cyanobacteria "Synechococcus" population on a map or over time

    ```r
    > stat <- get.stat.table() # to load the aggregate statistics
    > plot.map(stat, pop='synecho', param='abundance') 
    > plot.time(stat, pop='synecho', param='abundance')
    ```

    But you can plot any parameter/population, just make sure their name match the one in the 'stat' table... 

    FYI, type `colnames(stat)` to know which parameters are available in the 'stat' table,  and `unique(stat$pop)` to know the name of the different populations.

# ReAnalyze previous files

If you changes the filter parameters and need to reanalyze previous files according to these new parameters, use:

`> rerun.filter(start.day, start.timestamp, end.day, end.timestamp)`

where `start.day` and `end.day` represent the folder name (year_julianday) and `start.timestamp` and `end.timestamp` the file name (ISO8601) of the first and last file you want to reanalyze. This function will update the 'opp' table in the database, and also update the 'vct' and 'stats' table.

If you changes the gating parameters and need to reanalyze previous files according to these new parameters, use:

`> rerun.gating(start.day, start.timestamp, end.day, end.timestamp)`

This function will update the 'vct' and 'stats' table.
