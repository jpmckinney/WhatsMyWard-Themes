# coding: utf-8
require 'rubygems'
require 'bundler/setup'

desc %q(Build "What's My Ward?" themes for all municipal councils in Represent)
task :build do
  require 'fileutils'
  require 'json'
  require 'open-uri'

  require 'color-generator'
  require 'nokogiri'
  require 'ruby-progressbar'

  BASE_URL = 'http://represent.opennorth.ca'

  # @param [String] endpoint an API endpoint
  # @return [Hash] the JSON response
  def load_json(endpoint)
    JSON.load(open(BASE_URL + endpoint))['objects']
  end

  generator = ColorGenerator.new saturation: 0.5, lightness: 0.75

  load_json('/representative-sets/?limit=100').select do |representative_set|
    # Select only municipalities...
    representative_set['name'][/Town|City/] &&
    # ... that have subdivisions.
    !representative_set['related']['boundaries_url'].empty?
  end.each do |representative_set|
    puts "Building #{representative_set['name']}"
    boundaries_url = representative_set['related']['boundaries_url']

    # Create the theme directory.
    directory = "Themes/#{representative_set['name'].sub(/ (Town|City) Council/, '')}"
    FileUtils.mkdir_p directory

    # Get the list of representatives to process.
    representatives = load_json(representative_set['related']['representatives_url'] + '?limit=100').select do |representative|
      # Mayors will have a boundary_url of the entire city. We just want to get
      # the councillors, who represent wards.
      representative['related']['boundary_url'].include? boundaries_url
    end

    progressbar = ProgressBar.create format: '%a |%B| %p%% %e', length: 80, smoothing: 0.5, total: representatives.size
    representatives.each_with_index do |representative|
      progressbar.increment

      # Get the KML from Represent.
      kml = Nokogiri::XML(open(BASE_URL + representative['related']['boundary_url'] + 'simple_shape?format=kml')) do |config|
        config.default_xml.noblanks
      end

      # Prepare the JSON data for the description field.
      data = {}

      # Add basic attributes.
      { 'name'           => 'Councillor',
        'party_name'     => 'Party',
        'photo_url'      => 'Photo',
        'email'          => 'Email',
        'url'            => 'Website',
        'personal_url'   => 'Personal Website',
      }.each do |from,to|
        unless representative[from].nil? || representative[from].empty?
          data[to] = representative[from]
        end
      end

      # Add offices.
      representative['offices'].each do |office|
        { 'tel' => 'Phone',
          'fax' => 'Fax',
          'postal' => 'Address',
        }.each do |from,to|
          unless office[from].nil? || office[from].empty?
            data[to] = office[from]
          end
        end
      end

      # Add tags and attributes to the KML.
      slug = representative['district_name'].downcase.gsub(/[()]/, '').gsub(/\W/, '_')
      color = generator.create.upcase

      document = kml.at_css 'Document'
      document.add_child "<name>#{slug}.kml</name>"
      document.add_child %(<Style id="Style"><LineStyle><color>#{color}</color></LineStyle><PolyStyle><color>#{color}</color></PolyStyle></Style>)

      placemark = kml.at_css 'Placemark'
      placemark['id'] = slug
      placemark.add_child "<description><![CDATA[\n#{JSON.generate(data, object_nl: "\n")}\n]]></description>"
      placemark.add_child '<styleUrl>#Style</styleUrl>'

      # Write the KML file.
      File.open("#{directory}/#{slug}.kml", 'w') do |f|
        f.write kml.to_xml(indent: 2)
      end
    end
  end
end

desc %q(Delete all "What's My Ward?" themes)
task :clean do
  FileUtils.rm_r 'Themes', secure: true
end

