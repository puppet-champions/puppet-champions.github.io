require 'erb'
require 'base64'
require 'httparty'
require 'json'
require 'octokit'


REPO = 'puppet-champions/puppet-champions.github.io'
NIMBLE_TOKEN = ENV['NIMBLE_TOKEN']
GITHUB_TOKEN = ENV['GITHUB_TOKEN'] || `git config --global github.token`.chomp
@errorlevel  = 0

############################### Nimble functions ###############################
def nimble_id_map(name)
  ['none', 'Extraordinary Puppeteer', 'Champion'].index(name).to_s
end

def nimble_group(name)
  if NIMBLE_TOKEN.nil?
    puts "You need to generate a Nimble token:"
    puts "\t * http://support.nimble.com/en/articles/822159-generate-an-api-key-to-access-the-nimble-api"
    puts
    puts "Export that as the `NIMNLE_TOKEN` environment variable."
    exit 1
  end

  query = {
    "query"  => {"custom_fields" => {"Puppet Champion Status" => {"is" => nimble_id_map(name)}}}.to_json,
    "fields" => "email,GitHub Username",
  }

  response = HTTParty.get('https://app.nimble.com/api/v1/contacts',
                headers: {"Authorization" => "Bearer #{NIMBLE_TOKEN}"},
                query:   query,
  )

  raise response.body unless response.success?

  data = JSON.parse(response.body)
  without, with = data['resources'].partition {|item| item['fields']['GitHub Username'].nil?}

  unless without.empty?
    puts 'The following Nimble accounts have no GitHub username associated:'
    without.each do|item|
      puts "    ↳ #{item['fields']['email'].first['value']}"
      @errorlevel += 1
    end
  end

  # now return the ones we do know.
  with.map {|item| item['fields']['GitHub Username'].first['value'] }
end
############################ End Nimble functions ##############################


############################### GitHub functions ###############################
def client
  return @client if @client

  if GITHUB_TOKEN.empty?
    puts "You need to generate a GitHub token:"
    puts "\t * https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line"
    puts "\t * git config --global github.token <token>"
    puts
    puts "Export that as the `GITHUB_TOKEN` environment variable or put it in your ~/.gitconfig."
    exit 1
  end

  begin
    @client = Octokit::Client.new(:access_token => GITHUB_TOKEN)
  rescue => e
    puts "Github login error: #{e.message}"
    exit 1
  end
  @client
end

def get_file(path)
  begin
    Base64.decode64(client.contents(REPO, path: path).content)
  rescue Octokit::NotFound => e
    puts "File not found: #{path}"
    ''
  end
end

def create_profile(member, path)
  profile = ERB.new(get_file('_template.erb')).result(binding)
  welcome = ERB.new(get_file('_welcome.erb')).result(binding)
  subject = "Creating profile for #{member[:login]}"
  master  = 'heads/master'
  branch  = "heads/#{member[:login]}"

  begin
    master_sha    = client.ref(REPO, 'heads/master').object.sha
    branch_sha    = client.create_ref(REPO, branch, master_sha).object.sha
    base_tree_sha = client.commit(REPO, master_sha).commit.tree.sha
    blob_sha      = client.create_blob(REPO, Base64.encode64(profile), 'base64')
    new_tree_sha  = client.create_tree(REPO,
                                        [ { :path => "#{path}/#{member[:login]}.md",
                                            :mode => '100644',
                                            :type => 'blob',
                                            :sha  => blob_sha
                                          }
                                        ],
                                        {:base_tree => base_tree_sha }
                                      ).sha

    new_commit_sha = client.create_commit(REPO, subject, new_tree_sha, branch_sha).sha
    updated_ref    = client.update_ref(REPO, branch, new_commit_sha)
    pull_request   = client.create_pull_request(REPO, 'master', member[:login], subject, welcome)

    client.request_pull_request_review(REPO, pull_request[:number], reviewers: [member[:login]] )

  rescue Octokit::UnprocessableEntity => e
    if e.message.match /Reference already exists/
      puts "    ↳ Branch for #{member[:login]} already exists."
    else
      puts e.message
    end
    @errorlevel += 1
  end
end

def move_profile(member, source, dest)
  subject = "Moving profile for #{member[:login]}"
  master  = 'heads/master'
  branch  = "heads/#{member[:login]}"
  source  = "#{source}/#{member[:login]}.md"
  dest    = "#{dest}/#{member[:login]}.md"

  begin
    master_sha   = client.ref(REPO, 'heads/master').object.sha
    branch_sha   = client.create_ref(REPO, branch, master_sha).object.sha
    base_tree    = client.tree(REPO, master_sha, recursive: true).tree
    changed_tree = base_tree.reject { |blob| blob.type == 'tree' }
    changed_item = changed_tree.find {|blob| blob.path == source }

    # now finally rename the file!
    changed_item.path = dest

    # we need hashes and to clean up the elements
    changed_tree.map!(&:to_hash).each { |blob| blob.delete(:url) && blob.delete(:size) }

    new_tree_sha   = client.create_tree(REPO, changed_tree).sha
    new_commit_sha = client.create_commit(REPO, "Rename #{source} to #{dest}", new_tree_sha, master_sha).sha
    updated_ref    = client.update_ref(REPO, branch, new_commit_sha)
    pull_request   = client.create_pull_request(REPO, 'master', member[:login], subject, subject)

    # no need for a PR review, the CODEOWNERS should do that for us
    #client.request_pull_request_review(REPO, pull_request[:number], reviewers: '@puppet-champions/admins' )

  rescue Octokit::UnprocessableEntity => e
    if e.message.match /Reference already exists/
      puts "    ↳ Branch for #{member[:login]} already exists."
    else
      puts e.message
    end
    @errorlevel += 1
  end
end

def delete_profile(filename)
  file_sha   = client.contents(REPO, path: filename).sha
  delete_sha = client.delete_contents(REPO, filename, "Deleting #{filename}", file_sha)
end
############################ End GitHub functions ##############################


task :default do
  puts 'This rake task just automates the management of Champion profile pages.'
  puts 'It will create/remove/move profiles as markdown pages as users are updated'
  puts 'in Nimble.'
  puts
  puts 'The user is review-requested when necessary with instructions on how to'
  puts 'update and approve their profile.'
  puts
  puts "This sync task will run weekly, but if you'd like you can force an update manually."
  puts 'Simply run `rake sync` at the command line each time you update membership.'
  puts
  system("rake -T")
end


desc 'Sychronize profiles to match team membership'
task :sync do
  begin
    puppeteer_profiles = client.contents(REPO, :path => '_puppeteers').map {|obj| obj[:path] }.select {|f| f.end_with? '.md' }
    champion_profiles  = client.contents(REPO, :path => '_champions').map {|obj| obj[:path] }.select {|f| f.end_with? '.md' }
  rescue Octokit::NotFound => e
    puts "Path does not exist: #{e.message}"
    exit 1
  end

  puppeteers = nimble_group('Extraordinary Puppeteer')
  champions  = nimble_group('Champion')

  puppeteers.each do |login|
    member  = client.user(login)
    profile = "#{login}.md"

    next if puppeteer_profiles.include? "_puppeteers/#{profile}"

    if champion_profiles.include? "_champions/#{profile}"
      champion_profiles.delete("_champions/#{profile}")
      puts "Moving #{profile} from _champions to _puppeteers."
      move_profile(member, '_champions', '_puppeteers')
    else
      puts "Creating Puppeteer profile: #{profile}"
      create_profile(member, '_puppeteers')
    end
  end

  champions.each do |login|
    member  = client.user(login)
    profile = "#{login}.md"

    next if champion_profiles.include? "_champions/#{profile}"

    if puppeteer_profiles.include? "_puppeteers/#{profile}"
      puppeteer_profiles.delete("_puppeteers/#{profile}")
      puts "Moving #{profile} from _puppeteers to _champions."
      move_profile(member, '_puppeteers', '_champions')
    else
      puts "Creating Champion profile: #{profile}"
      create_profile(member, '_champions')
    end
  end

  # now clean up leftovers
  (puppeteer_profiles - puppeteers.map {|member| "_puppeteers/#{member}.md" }).each do |path|
    puts "Removing #{path}"
    delete_profile(path)
  end

  (champion_profiles - champions.map {|member| "_champions/#{member}.md" }).each do |path|
    puts "Removing #{path}"
    delete_profile(path)
  end

  unless @errorlevel == 0
    puts '-----------------------------------------'
    abort "There were #{@errorlevel} sync warnings."
  end
end

