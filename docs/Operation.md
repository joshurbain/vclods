# Operation -- Dot Extensions
## Location
Name | Description
-----|------------
ANY  | Normal processing. takes stdin (or the contents of the file), does something, and leaves output for the next thing (or logging).
TERM | Ends the .extension chain by redirecting stdin somewhere else
PIPE | Pass Through: Must be in the middle of a .extension chain
WRAP | Runs another .extension tree as a child
FLOW | Controls the flow of the application, allowing for branches

## Option
Many extensions have the abilty to accept one and only one option in the .extension chain. This is signified as `<base_filename>.<other_optional_extensions>.<extension>-<option>.<more_optional_extensions>`. All other behavior modification is handled with Configuration Variables. Options allow 2 of the same extension to be present in the same .extension chain while exhibiting different behavior. Most often, the option value is a file name (due to the nature of the .extension chain, these filenames may not themselves have a .extension). If another file is accepted, the default for the option will be `$base_filename`
## Extensions
Name | Location | Option Default | Description
-----|----------|----------------|------------
add|PIPE|$base_filename|Prepend a file's contents to the extension pipe
awk|PIPE|Used, No Default|Run awk program
batch|PIPE|Used, No Default|aggregate input into batch SQL statements
curl|ANY|Not Used|cURL wrapper with optional batching using .jq
diff|TERM|Not Used|diff extension pipe with a "static" file (or generate output based on provided operation)
dir|WRAP|${DIR_EXT_START:?}|Run a subdirectory where the last directory name holds the .extension on how to process the files (the files may not have .extensions of their own).
dst|ANY|Not Used|Run a SQL script with the secondary connection (configured with VCLOD_DST_)
email|TERM|Not Used|email extension pipe to SUPPORT_EMAIL either as an attachment or inline.
end|PIPE|$base_filename|Postpend a file's contents to the extension pipe
err|TERM|Not Used|Everything to this point is an error. Great for Data tests.
etl|PIPE|Used, No Default|Preform advanced ETL operations based on a temp table definition with structured comments. Should be followed by .batch.(sql|dst).
g|WRAP|Used, No Default|Guard another extension (like in .g-jq) by saving its inputs on error alongside normal logging
jq|PIPE|Used, No Default|Run stdin through the jq utility to parse JSON
litsh|PIPE|$HEREDOC_DELIMITER|literate source: inverts code and comments allowing ksh (ie, process and variable subsitition and expansion) inside prose. Use with caution!
null|TERM|Not Used|dump stdout to /dev/null, effectively silencing output.
out|TERM|$base_filename|Write to file; stop
outa|TERM|$base_filename|Append to file; stop
py|ANY|Used, No Default|Run either stdin or ext_opt file (with any optional ending) as python3
sh|ANY|Used, No Default|Source a ksh script
shard|FLOW|sh|Running a stdin with the given operation $SHARD_EXT_COUNT times. Optional run batches in series, sleep a set interval, or go full for parallelity.
shebang|ANY|Not Used|Respect script's first line shebang (default: source in ksh just like .sh)
slack|TERM|Not Used|push stdin to slack channel. Will propogate all data on.
split|FLOW|sh|Split records into batched operations... running the same operation for every $SPLIT_EXT_COUNT rows
sql|ANY|Not Used|Run a SQL script with default connection (configured with VCLOD_SRC_)
tee|PIPE|$base_filename|Route output to file and the remaining extension pipe
teea|PIPE|$base_filename|Route output to file (appended) and the remaining extension pipe
vfs|WRAP|Not Used|virtual file system: use #fifo to generate virtual files and #run to use those files inside a VCLOD context. You can only use each virtual file once per run. Helps with large composite programs with many small files. Especially useful for running oneoff scripts from stdin
while|FLOW|sh|Keep running stdin with the given operation until it returns a non-zero exit code (and optionally stops producing output)
wrap|PIPE|$base_filename|Wrap stdin with files like .add and .end, but using filenames ending in __beg and __end
