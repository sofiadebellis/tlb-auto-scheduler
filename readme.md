## How to use the TLB auto scheduler

This document describes how to use a small program I wrote which automatically computes class schedules. It is specifically designed for courses which match COMP1521’s TLBs: 1 hour of tutorial with one tutor, followed by 2 hours of lab with that same tutor as well as an additional lab assistant. The underlying backend solver works with other class structures but you’ll need to modify some of the code to get that to work.

The program is designed for larger courses where manually making a schedule is time-consuming. For smaller courses with fewer classes it’s probably a better idea to just do this manually.

There are two main inputs that you need to provide to the program: a list of classes that it will be scheduling for, and a list of instructors that it can use. These two inputs are specified as TSV (tab-separated value) files, and are easiest to construct by preparing them in a spreadsheet and then directly copy-pasting them into a file.

I recommend constructing these TSVs using Google Sheets by copying this [template](https://docs.google.com/spreadsheets/d/1u3LQSTD8C5aI3aU66kSmWz_p93r2dcBcRNVL3Nc8Nus/edit?usp=sharing).


1. Get the code
2. Grab a copy of the code:

```bash
$ git clone https://github.com/sofiadebelis/tlb-auto-scheduler.git
$ cd tlb-auto-scheduler
```

3. Create a directory which will contain the class/instructor details for your course. You can name it however you want, but here I’ll name it `config.25T1`:

```bash
$ mkdir config.25T1
```

4. Next, copy in the sample costs configuration.

```bash
$ cp costs.example.toml config.25T1/costs.toml
```

### Classes

Into the Classes spreadsheet copy-paste all of the classutil TLBs for your course. Only copy the TLBs, not the LECs or CRS. Leave row 1 (the header row) of the template spreadsheet intact. It should look like this:

Go through and remove all of the Tent and Closed TLBs by deleting those rows. Or if you want to schedule them because you know they’re going to open, change the status to Open or Full.

If you don’t want the TLB scheduler to assign a tut/lab for a TLB, put a 1 in the ignore tut / ignore lab column for that row.

Create a file in the config directory you made previously (e.g. `config.25T1`) called `classes.tsv`. Copy-paste the entire classes table into the `classes.tsv` file. 

Make sure you’ve copied it as tab-separated.

### Instructors

Who you’re going to (potentially) hire is specified in the Instructors sheet.

For each instructor that you’ll be hiring you need to specify the following fields:

- zid: the instructor’s zid
- name: the instructor’s name
- minT: the minimum number of tutorials you want to the instructor to teach
- maxT: the maximum number of tutorials you want to the instructor to teach
- minA: the minimum number of lab assists you want to the instructor to teach
- maxA: the maximum number of lab assists you want to the instructor to teach
- minC: the minimum number of overall classes you want to the instructor to teach
- maxC: the maximum number of overall classes you want to the instructor to teach

You can ignore/delete the prevT and prevA columns (I use them to count the number of previous tutorials/labs the instructor has taught).

The combination of minT, maxT, minA, maxA, minC and maxC allow you to control how many and what type of classes are allocated to each tutor.

Additionally you can specify an ignore field. If this field is 1 then the instructor will not be used at all for scheduling.

You should make sure that these constraints are followed:

- The total of all the minT values should be below the number of tutorials.
- The total of all the maxT values should be above the number of tutorials.
- The total of all the minA values should be below the number of labs.
- The total of all the maxA values should be above the number of labs.
- The total of all the minC values should be below the number of classes (which is twice the number of TLBs).
- The total of all the maxC values should be above the number of classes (which is twice the number of TLBs)
- You should aim to have a bit of slack in the constraints, so that there’s a gap between the min and max totals. This allows for better scheduling.

On the instructors sheet in the template (to the right of the main table) there’s another table which will count up the min/max values. Make sure all the over/under cells are green. For this to work you need to manually set the #classes to the total number of classes (this won’t happen automatically). The in-sheet total calculation still counts ignored instructors, so if you’re using this table make sure to set the minT/maxT/minA/maxA/minC/maxC values of ignored instructors 0 to not accidentally count them.

The TLB auto scheduler itself will also perform these checks, as well as additional checks to make sure for each instructor the numbers make sense, like maxT should not be less than minT.

Once you’ve got the instructors, copy-paste the table, including the headers (but excluding the total table) into a file called instructors.tsv in the config directory you created.

### Running

You’ll need a recent-ish version of Rust (e.g. via rustup).

The first time you run the program it’ll need to download all the Talloc applications. To allow it do this grab a Talloc API JWT token (https://talloc.cse.unsw.edu.au/admin/api) and create a file called jwt (in your current working directory, not the config directory), containing your Talloc JWT. Subsequent runs will use the `talloc_cache.json` it creates in the config directory. Delete the `talloc_cache.json` file if you want an updated version of the Talloc applications.

To run the scheduler, use

```bash
$ cargo run --release -- config.25T1
```

(replacing `config.25T1` with what you called the configuration directory). The first time it’ll take a while to build, as well as a while to fetch all the Talloc applications.

Make sure to read any warnings it gives about bad constraints.

For a list of options run

```bash
$ cargo run --release -- --help
```

By default the scheduler only uses one thread, but you can exploit parallelism use the --cpus argument, e.g.

```bash
$ cargo run --release -- config.25T1 --cpus 8
```

As it runs and finds solutions it will create directories in the output directory. The latest subdirectory will contain the best result from the latest run. Adjust the `--total-attempts` to specify how many attempts it should make at finding a solution. Make `--num-rounds` larger to make the scheduler try more per attempt.

The first file to check in these directories is `solver_log.txt`. At the end it breaks down the ‘cost’ of the solution it found. A lower cost is better. If the cost consists of mostly AssignedPreferred with some AssignedPossible, then you’ve got (what the scheduler thinks) is a good solution. This means the total final cost will be less than around 50-100.

If it’s more there’s some issue, probably the problem is overly constrained and so there’s no good solution. Try decreasing minA, minT, minC or increasing maxA, maxT or maxC for a number of instructors. Sometimes some tutors only give very few available slots, and so it’s not possible to meet the minA/minT/minC requirements for them. Search for availabilities: in the `problem.txt` that gets created in the output directory.

Once you’re happy, copy the `solution.tsv` into a spreadsheet. Or if you’re in a rush you can use `scripts/give_admin_js.py` to generate some JavaScript that you can run when you’re on the Give Admin page which fills out the allocation for you.

### Misc

You can tweak the `costs.toml` to prioritise/deprioritise certain constraints.

If you put an `initial.tsv` file into the config directory which is a copy of a previous `solution.tsv` and set mismatched_initial_solution to a small number the solver will minimise the changes (and produce a `diff.txt`).

You can create Talloc application overrides in overrides.tsv in the config directory, to e.g. ensure a certain tutor isn’t scheduled for a certain class (or is only scheduled for a certain class). See `src/overrides.rs`.

`scripts/run_remote.py` has a script to run the solver on a bunch of lab machines. But it’s not super polished right now (and in my experience with COMP1521 running on a bunch of lab machines doesn’t help much compared to just running on my laptop).
