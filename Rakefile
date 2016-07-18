desc 'Start server localhost with watch mode on'
task :watch do
  exec jekyll_serve('-w')
end
task :default => :watch

desc 'Start server on localhost'
task :serve do
  exec jekyll_serve
end

desc 'Create new post'
task :new do
  print 'Title: '
  title = STDIN.gets.chomp
  filename = to_filename(title)
  path = "_posts/#{filename}"
  content = <<YML
---
layout: post
title: #{title}
---
YML

  unless File.exists?(path)
    File.write(path, content)
  end
end

def jekyll_serve(options = nil)
  # load local overrides in config
  ['jekyll serve --config _config.yml,_local_config.yml -I', options].compact.join(' ')
end

def to_filename(raw)
  prefix = Time.now.strftime '%Y-%m-%d'
  title = raw.downcase.gsub(' ', '-').tr('^-a-z0-9', '')
  "#{prefix}-#{title}.md"
end
