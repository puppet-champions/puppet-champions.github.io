require 'erb'
require 'base64'

# TODO: switch to DirectoryBot when that's completed

begin
  require 'octokit'
rescue LoadError => e
  puts "This requires the octokit gem"
  exit 1
end

TOKEN  = `git config --global github.token`.chomp
if TOKEN.empty?
  puts "You need to generate a GitHub token:"
  puts "\t * https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line"
  puts "\t * git config --global github.token <token>"
  exit 1
end

begin
  client = Octokit::Client.new(:access_token => TOKEN)
  client.user.login
  client.auto_paginate = true
rescue => e
  puts "Github login error: #{e.message}"
  exit 1
end

org  = 'puppet-champions'
repo = 'puppet-champions/puppet-champions.github.io'

admins     = client.team_members(client.org_teams(org).find {|t| t[:name] == 'Admins'}[:id])
puppeteers = client.team_members(client.org_teams(org).find {|t| t[:name] == 'Extraordinary Puppeteers'}[:id])
champions  = client.org_members(org)

puppeteers.reject! do |member|
  admins.find {|adm| member.id == adm.id }
end

champions.reject! do |member|
  admins.find {|adm| member.id == adm.id } or puppeteers.find {|ppt| member.id == ppt.id }
end

begin
  puppeteer_profiles = client.contents(repo, :path => '_puppeteers').map {|obj| obj[:path] }.select {|f| f.end_with? '.md' }
  champion_profiles  = client.contents(repo, :path => '_champions').map {|obj| obj[:path] }.select {|f| f.end_with? '.md' }
rescue Octokit::NotFound => e
  puts "Path does not exist: #{e.message}"
  exit 1
end

def get_file(client, repo, path)
  begin
    Base64.decode64(client.contents(repo, path: path).content)
  rescue Octokit::NotFound => e
    puts "File not found: #{path}"
    ''
  end
end

def create_profile(client, repo, member, path)
  member  = client.user(member[:id]) # get the full user object
  profile = ERB.new(get_file(client, repo, '_template.erb')).result(binding)
  welcome = ERB.new(get_file(client, repo, '_welcome.erb')).result(binding)
  subject = "Creating profile for #{member[:login]}"
  master  = 'heads/master'
  branch  = "heads/#{member[:login]}"

  begin
    master_sha    = client.ref(repo, 'heads/master').object.sha
    branch_sha    = client.create_ref(repo, branch, master_sha).object.sha
    base_tree_sha = client.commit(repo, master_sha).commit.tree.sha
    blob_sha      = client.create_blob(repo, Base64.encode64(profile), 'base64')
    new_tree_sha  = client.create_tree(repo,
                                        [ { :path => "#{path}/#{member[:login]}.md",
                                            :mode => '100644',
                                            :type => 'blob',
                                            :sha  => blob_sha
                                          }
                                        ],
                                        {:base_tree => base_tree_sha }
                                      ).sha

    new_commit_sha = client.create_commit(repo, subject, new_tree_sha, branch_sha).sha
    updated_ref    = client.update_ref(repo, branch, new_commit_sha)
    pull_request   = client.create_pull_request(repo, 'master', member[:login], subject, welcome)

    unless member[:login] == client.login
      client.request_pull_request_review(repo, pull_request[:number], reviewers: [member[:login]] )
    end

  rescue Octokit::UnprocessableEntity => e
    if e.message.match /Reference already exists/
      puts "    ↳ Branch for #{member[:login]} already exists."
    else
      puts e.message
    end
  end
end

def move_profile(client, repo, member, source, dest)
  member  = client.user(member[:id]) # get the full user object
  welcome = ERB.new(get_file(client, repo, '_welcome.erb')).result(binding)
  subject = "Moving profile for #{member[:login]}"
  master  = 'heads/master'
  branch  = "heads/#{member[:login]}"
  source  = "#{source}/#{member[:login]}.md"
  dest    = "#{dest}/#{member[:login]}.md"

  begin
    master_sha   = client.ref(repo, 'heads/master').object.sha
    branch_sha   = client.create_ref(repo, branch, master_sha).object.sha
    base_tree    = client.tree(repo, master_sha, recursive: true).tree
    changed_tree = base_tree.reject { |blob| blob.type == 'tree' }
    changed_item = changed_tree.find {|blob| blob.path == source }

    # now finally rename the file!
    changed_item.path = dest

    # we need hashes and to clean up the elements
    changed_tree.map!(&:to_hash).each { |blob| blob.delete(:url) && blob.delete(:size) }

    new_tree_sha   = client.create_tree(repo, changed_tree).sha
    new_commit_sha = client.create_commit(repo, "Rename #{source} to #{dest}", new_tree_sha, master_sha).sha
    updated_ref    = client.update_ref(repo, branch, new_commit_sha)
    pull_request   = client.create_pull_request(repo, 'master', member[:login], subject, welcome)

    unless member[:login] == client.login
      client.request_pull_request_review(repo, pull_request[:number], reviewers: [member[:login]] )
    end

  rescue Octokit::UnprocessableEntity => e
    if e.message.match /Reference already exists/
      puts "    ↳ Branch for #{member[:login]} already exists."
    else
      puts e.message
    end
  end
end

def delete_profile(client, repo, filename)
  file_sha   = client.contents(repo, path: filename).sha
  delete_sha = client.delete_contents(repo, filename, "Deleting #{filename}", file_sha)
end

task :default do
  puts 'This rake task just automates the management of Champion profile pages.'
  puts 'It will create profiles when users are added to the org, and delete them'
  puts 'when users are removed. It will also move profiles between the two tier'
  puts 'when user team membership changes. The user is review-requested with'
  puts 'instructions on how to update and approve their profile.'
  puts
  puts 'Simply run `rake sync` at the command line each time you update membership.'
  puts
  system("rake -T")
end

desc 'Sychronize profiles to match team membership'
task :sync do
  puppeteers.each do |member|
    login   = member[:login]
    profile = "#{login}.md"

    next if puppeteer_profiles.include? "_puppeteers/#{profile}"

    if champion_profiles.include? "_champions/#{profile}"
      champion_profiles.delete("_champions/#{profile}")
      puts "Moving #{profile} from _champions to _puppeteers."
      move_profile(client, repo, member, '_champions', '_puppeteers')
    else
      puts "Creating Puppeteer profile: #{profile}"
      create_profile(client, repo, member, '_puppeteers')
    end
  end

  champions.each do |member|
    login   = member[:login]
    profile = "#{login}.md"

    next if champion_profiles.include? "_champions/#{profile}"

    if puppeteer_profiles.include? "_puppeteers/#{profile}"
      puppeteer_profiles.delete("_puppeteers/#{profile}")
      puts "Moving #{profile} from _puppeteers to _champions."
      move_profile(client, repo, member, '_puppeteers', '_champions')
    else
      puts "Creating Champion profile: #{profile}"
      create_profile(client, repo, member, '_champions')
    end
  end

  # now clean up leftovers
  (puppeteer_profiles - champions.map {|m| "_puppeteers/#{m[:login]}.md" }).each do |path|
    puts "Removing #{path}"
    delete_profile(client, repo, path)
  end

  (champion_profiles - champions.map {|m| "_champions/#{m[:login]}.md" }).each do |path|
    puts "Removing #{path}"
    delete_profile(client, repo, path)
  end
end
