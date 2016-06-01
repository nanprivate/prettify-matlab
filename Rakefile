#!/usr/bin/env ruby

require "tempfile"
require "fileutils"

# default task
task :default => "SO:build"

# source files to be processed
SOURCES = %w[
    lang-matlab.js
    prettify-matlab.user.js
    prettify-mathworks-answers.user.js
    prettify-mathworks-fileexchange.user.js
    prettify-mathworks-examples.user.js
    switch-lang.user.js
]
DEST = %w[
    dist/js/full
    dist/userscripts
    dist/userscripts
    dist/userscripts
    dist/userscripts
    dist/userscripts
]

namespace :SO do
    desc "Builds code-prettify extension and userscripts"
    task :build do
        SOURCES.zip(DEST).each do |name, dest|
            # create destination directory if needed
            unless File.directory?(dest)
                FileUtils.mkdir_p(dest)
            end

            # process file
            source = File.open("src/#{name}", "rb")
            target = File.open("#{dest}/#{name}", "wb")
            begin
                puts "Building #{target.path}"
                process_file(source, target)
            ensure
                source.close
                target.close
            end
        end
    end

    desc "Run watchr"
    task :watchr do
        require "rubygems"
        require "watchr"
        script = Watchr::Script.new
        all_files = [Dir["src/*.js"], Dir["src/*.css"]].join("|")
        script.watch(all_files) do |file|
            Rake::Task["SO:build"].execute
        end
        controller = Watchr::Controller.new(script, Watchr.handler.new)
        controller.run
    end
end

# process file by parsing simple templating instructions recursively
# Adapted from: https://github.com/mislav/user-scripts/blob/master/Thorfile
def process_file(source, target)
    # read source line-by-line.
    for line in source
        case line
        # match instructions: INSERT/RENDER, QUOTED, CONCATED
        when %r{^(\s*)//=(INSERT|RENDER)_FILE(_QUOTED)?(_CONCATED)?=\s+(.*)$}
            # get indentation, process mode, quoted lines, concatenated lines,
            # and name of file to insert
            indentation, processMode, doQuote, doConcat, filename =
                $1, $2, $3, $4, $5

            # check file exists (filename is relative to source)
            filename = File.join(File.dirname(source.path), filename.strip)
            filename = File.expand_path(filename)
            if not File.exist?(filename)
                # warn user and pass line unchanged
                puts "WARNING: file not found #{filename}"
                target << line
                next
            end

            # INSERT vs. RENDER
            if processMode == "RENDER"
                # render it into a temp file
                template = File.open(filename, "rb")
                tmp = Tempfile.new(File.basename(filename))
                tmp.binmode  # force binary mode to disable newline conversion
                begin
                    process_file(template, tmp)
                    # switch to using the processed file
                    filename = tmp.path
                ensure
                    template.close
                    tmp.close
                end
            end

            # concatenate all file lines by "|" as one string
            if doConcat
                # read file lines and join by "|"
                insert_line = File.readlines(filename).map(&:rstrip).join("|")

                # write concatenated line
                target << indentation  # keep same level of indentation
                target << (doQuote ? quote_string(insert_line,false) : insert_line)
                target << "\n"         # reinsert new line at the end

            # write file lines one-by-one
            else
                file = File.open(filename, "rb")
                for insert_line in file
                    target << indentation if doQuote or
                        not insert_line.to_s.strip.empty?
                    target << (doQuote ? quote_string(insert_line) : insert_line)
                end
                file.close
            end

            if processMode == "RENDER"
                # delete temporary file
                tmp.unlink
            end

        # else pass the line unchanged to target
        else
            target << line
        end
    end
end

# returns: 'str',
def quote_string(str, doTrailComma=true)
    ret = "'" + str.gsub(/(\n)$/,"") + "'"
    ret += "," if doTrailComma
    ret += $1 if not $1.nil?
    ret
end
