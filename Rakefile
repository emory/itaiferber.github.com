require 'rubygems'
require 'cloudfiles'
require 'uri'

# -----------------------------------------------------------------------------
#  => Post Model
# -----------------------------------------------------------------------------
# A class representing a post (comprised of YAML frontmatter and string contents).
class Post
	# Post content.
	attr_accessor :frontmatter
	attr_accessor :content

	# Loading from a file.
	def Post.load(path)
		if File.exist?(path) then content = File.read(path) else raise "No file at given path." end
		frontmatter = {}

		# Extract the YAML frontmatter.
		match = /[-]{3}\n(?<frontmatter>((.+: ?.*)?\n)*)[-]{3}\n/.match(content)
		if match != nil
			# Remove the frontmatter from the content.
			content.sub!("---\n#{match[:frontmatter]}---\n", "")
			if match[:frontmatter].length > 0
				# Store the frontmatter.
				match[:frontmatter].each_line do |line|
					if /(?<variable>.+?): ?(?<content>.+)/.match(line)
						frontmatter[$~[:variable].intern] = $~[:content].strip
					end
				end
			end
		end

		# Return the generated post.
		Post.new(frontmatter, content)
	end

	# Initialization.
	def initialize(frontmatter = {}, content = "")
		raise ArgumentException unless frontmatter.class is Hash and content.class is String
		@frontmatter = frontmatter
		@content = content
	end

	# Writing to a file.
	def write(path)
		File.open(File.expand_path(path), "w") {|file| file.print self.to_s}
	end

	# Conversion to string.
	def to_s
		representation = "---" << "\n"
		@frontmatter.each_key {|key| representation << "#{key}: #{@frontmatter[key]}\n"}
		representation << "---" << "\n"
		representation << @content << "\n"
	end
end

# -----------------------------------------------------------------------------
#  => Command Line Interaction
# -----------------------------------------------------------------------------
# A module for interacting with the user. String interpolation is used for type safety.
module User
	# Print the given message and get a response.
	def User.input(message = "")
		print "#{message}"
		STDIN.gets.chomp
	end

	# Print a message and ask the user for a yes or no message.
	# Returns a boolean.
	def User.confirm(message = "")
		response = input("#{message} [y/n] ") while response != 'y' and response != 'n'
		response == 'y'
	end

	# Asks the user to choose from the given options.
	# Returns the index of the choice or -1 if the user has opted to cancel.
	def User.choose(message = "", options = [])
		# No options mean cancel.
		return -1 if options.empty?

		# Print the options.
		options.each_index {|index| puts "#{index + 1}. #{options[index]}"}

		# Loop until a valid response is returned and return the appropriate index.
		response = input("#{message} [#{options.length > 1 ? Range.new(1, options.length) : 1}/c] ") while !(response == 'c' or (response.to_i > 0 and response.to_i <= options.length))
		response == 'c' ? -1 : response.to_i - 1
	end
end

# -----------------------------------------------------------------------------
#  => Helper Methods
# -----------------------------------------------------------------------------
# A module for listing helper methods.
module Helper
	# List of all posts in '_posts' directory.
	def Helper.posts
		Dir.entries("_posts").reject {|file| /^((?!\.)(.*(\.md)*)*)*$/.match(file) == nil}
	end

	# List of unpublished drafts.
	def Helper.unpublished
		Helper.posts.reject {|file| Post.load("_posts/#{file}").frontmatter[:published] == "true"}
	end

	# List of published posts.
	def Helper.published
		Helper.posts.reject {|file| Post.load("_posts/#{file}").frontmatter[:published] == "false"}
	end
end

# -----------------------------------------------------------------------------
#  => Tasks
# -----------------------------------------------------------------------------
# Set the default task.
task :default => ['draft:list']

# Tasks available for working with unpublished drafts.
namespace :draft do
	desc "Lists available drafts."
	task :list do
		# Get a list of unpublished drafts and print it.
		(files = Helper.unpublished) and (abort 'No drafts.' if files.empty?)
		files.each_index {|index| puts "#{index + 1}. #{files[index]}"}
	end

	desc "Creates a new draft with the given title and tags."
	task :new, [:title, :tags] do |t, args|
		# Set up default parameters.
		args.with_defaults(:title => "", :tags => "")
		(title = args[:title]) and (abort "Can't create post without title!" if title == "")

		# Check if post with same name already exists.
		filename = "#{title.downcase.gsub(/[^\w ]/, '').gsub(' ', '-')}"
		expression = Regexp.compile(filename)
		abort "'#{title}' already exists." if Helper.posts.reject{|file| expression.match(file) == nil}.size > 0

		# Create new post with parameters.
		date = Time.now
		filename = "_posts/#{date.strftime("%Y-%m-%d")}-#{filename}-draft.md"
		frontmatter = {:layout => "post", :title => title, :date => "#{date.strftime("%Y-%m-%d %H:%M")}", :tags => "[#{args[:tags].split(/[^\w-]/).reject{|tag| tag == ""}.join(', ')}]", :published => "false"}

		# Write post to file and open it.
		Post.new(frontmatter).write(filename)
		`open #{filename}`
	end

	desc "Opens an existing draft matching the given expression for editing."
	task :edit, [:expression] do |t, args|
		# By default, list all available drafts.
		args.with_defaults(:expression => ".md")

		# Reject posts that don't match the expression.
		(files = Helper.unpublished.reject {|file| /^((?!\.)(.*(#{args[:expression].gsub('.', '\\.')})*)*)*$/.match(file) == nil}) and (abort 'No drafts.' if files.empty?)

		# Let the user edit a file.
		(index = User.choose("Which draft would you like to edit?", files)) and (abort 'Canceled.' if index == -1)
		`open _posts/#{files[index]}`
	end

	desc "Deletes a draft matching the given expression."
	task :delete, [:expression] do |t, args|
		# By default, list all drafts.
		args.with_defaults(:expression => ".md")

		# Generate a list of files matching the expression.
		(files = Helper.unpublished.reject {|file| /^((?!\.)(.*(#{args[:expression].gsub('.', '\\.')})*)*)*$/.match(file) == nil}) and (abort 'No drafts.' if files.empty?)
		index = User.choose("Which draft would you like to delete?", files)
		abort 'Canceled.' if index == -1

		# Delete the draft.
		`rm _posts/#{files[index]}`
	end

	desc "Publishes a post matching the given expression."
	task :publish, [:expression] do |t, args|
		# By default, list all drafts.
		args.with_defaults(:expression => ".md")

		# Reject drafts that don't match the description.
		(files = Helper.unpublished.reject {|file| /^((?!\.)(.*(#{args[:expression].gsub('.', '\\.')})*)*)*$/.match(file) == nil}) and (abort 'No drafts.' if files.empty?)

		# Choose a draft.
		(index = User.choose("Which draft would you like to publish?", files)) and (abort 'Canceled.' if index == -1)

		# Update draft frontmatter.
		date = Time.now
		post = Post.load("_posts/#{files[index]}")
		post.frontmatter[:layout] = "post"
		post.frontmatter[:date] = "#{date.strftime("%Y-%m-%d %H:%M")}"
		post.frontmatter[:published] = "true"

		# Search for local resource URLs. Upload to Rackspace Cloud Files using API.
		resourceRegexp = /\[(?<title>.+)\]\(file:\/\/localhost(?<path>.+)\)/
		if resourceRegexp.match post.content then
			# Connect, pulling API key from OS X keychain.
			cloudFiles = CloudFiles::Connection.new(:username => 'itaiferber', :api_key => `security find-internet-password -g -s rackspace 2>&1 >/dev/null`[/^password: "(.*)"$/, 1])
			container = cloudFiles.container('itaiferber.net')

			# Check for continual matching.
			while resourceRegexp.match post.content
				title = $~['title']
				path = $~['path']

				# Check to see that local file really exists.
				if File.exists? path then
					# Generate a filename for the uploaded file in the form 'post-title-escaped-resource-name.extension'.
					filename = "#{post.frontmatter[:title]} #{/\/(.+\/)?(?<filename>.+\..+)/.match(path)['filename']}".downcase.gsub(/[^ \w.]+/, '').gsub(' ', '-')

					# Upload file contents.
					object = container.create_object filename
					object.write File.read path

					# Replace the local URL with the online URL.
					post.content.gsub!("file://localhost#{path}", object.public_url)
				else
					puts "No file at '${path}'."
				end
			end
		end

		# Make the draft a real post.
		post.write("_posts/#{files[index].sub("-draft.md", ".md")}")
		File.delete("_posts/#{files[index]}")
	end
end

# Tasks available for working with published posts.
namespace :post do
	desc "Lists published posts."
	task :list do
		(files = Helper.published) and (abort 'No posts.' if files.empty?)
		files.each_index {|index| puts "#{index + 1}. #{files[index]}"}
	end

	desc "Opens a published post for reading."
	task :open do
		# By default, list all posts.
		args.with_defaults(:expression => ".md")

		# Reject posts that don't match the expression.
		(files = Helper.published.reject {|file| /^((?!\.)(.*(#{args[:expression].gsub('.', '\\.')})*)*)*$/.match(file) == nil}) and (abort 'No posts.' if files.empty?)

		# Open the given post.
		(index = User.choose("Which post would you like to open?", files)) and (abort 'Canceled.' if index == -1)
		`open _posts/#{files[index]}`
	end

	desc "Unpublishes a post matching the given expression."
	task :unpublish, [:expression] do |t, args|
		# By default, list all posts.
		args.with_defaults(:expression => ".md")
		files = Helper.published.reject {|file| /^((?!\.)(.*(#{args[:expression].gsub('.', '\\.')})*)*)*$/.match(file) == nil}
		abort 'No posts.' if files.empty?

		# Choose a post.
		index = User.choose("Which post would you like to unpublish?", files)
		abort 'Canceled.' if index == -1

		# Update frontmatter.
		post = Post.load("_posts/#{files[index]}")
		post.frontmatter[:published] = "false"

		# Revert post to draft form.
		post.write("_posts/#{files[index].sub(".md", "-draft.md")}")
		File.delete("_posts/#{files[index]}")
	end
end

# Tasks available for working with the site itself.
namespace :site do
	desc "Start Jekyll in a local webserver."
	task :server do
		# Spawn child processes.
		compassPID = Process.spawn("compass watch")
		jekyllPID = Process.spawn("jekyll --auto --pygments --server")

		# Set up interrupt trap.
		trap("INT") {
			[compassPID, jekyllPID].each {|pid| Process.kill(9, pid) rescue Errno::ESRCH}
		}

		# Wait three seconds and open site.
		`sleep 3s && open http://24.146.156.205:4000/`
		[compassPID, jekyllPID].each {|pid| Process.wait(pid)}
	end
end
