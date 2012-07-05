# Encoding: UTF-8

# Landslide sunumlarını üretir ve indeksler
#
# Landslide yapılandırma dosyası (.cfg uzantılı) içeren her alt dizin bir
# sunumdur.  Her sunum için ilgili alt dizin ismiyle aynı isimde görevler
# tanımlanır.  Bu görevler sunum dizinlerinde dizin ismiyle aynı isimde html
# uzantılı sunumları üretir.

require 'pathname'
require '/var/lib/gems/1.8/gems/pythonconfig-1.0.1/lib/pythonconfig.rb'
require 'erb'

# Landslide yapılandırma dosyaları
CFGFILES       = FileList["[^_.]*/*.cfg"]
# İzin verilen en büyük resim boyutları
IMAGE_GEOMETRY = [ 733, 550 ]
# Öntanımlı sıra numarası
DEFAULT_ORDER  = 500
# index.html için şablon
TEMPLATE_INDEX = "_assets/index.erb"
# Bağımlılıklar için yapılandırmada hangi anahtarlara bakılacak
DEPEND_KEYS    = %w(source css js)
# Vara daima bağımlılık verilecek dosya/dizinler
DEPEND_ALWAYS  = %w(media)
# Sunum bilgileri
PRESENTATION   = {}
# Etiket bilgileri
TAG            = {}

class File
  @@absolute_path_here = Pathname.new(Pathname.pwd)
  def self.to_herepath(path)
    Pathname.new(File.expand_path(path)).relative_path_from(@@absolute_path_here).to_s
  end
  def self.to_filelist(path)
    File.directory?(path) ?
      FileList[File.join(path, '*')].select { |f| File.file?(f) } :
      [path]
  end
end

def png_comment(file, string)
  require 'chunky_png'
  require 'oily_png'

  image = ChunkyPNG::Image.from_file(file)
  image.metadata['Comment'] = 'raked'
  image.save(file)
end

def png_optim(file, threshold=40000)
  return if File.new(file).size < threshold
  sh "pngnq -f -e .png-nq #{file}"
  out = "#{file}-nq"
  if File.exist?(out)
    $?.success? ? File.rename(out, file) : File.delete(out)
  end
  # İşlendiğini belirtmek için not düş.
  png_comment(file, 'raked')
end

def jpg_optim(file)
  sh "jpegoptim -q -m80 #{file}"
  sh "mogrify -comment 'raked' #{file}"
end

def optim
  pngs, jpgs = FileList["**/*.png"], FileList["**/*.jpg", "**/*.jpeg"]

  # Optimize edilmişleri çıkar.
  [pngs, jpgs].each do |a|
    a.reject! { |f| %x{identify -format '%c' #{f}} =~ /[Rr]aked/ }
  end

  # Boyut düzeltmesi yap.
  (pngs + jpgs).each do |f|
    w, h = %x{identify -format '%[fx:w] %[fx:h]' #{f}}.split.map { |e| e.to_i }
    size, i = [w, h].each_with_index.max
    if size > IMAGE_GEOMETRY[i]
      arg = (i > 0 ? 'x' : '') + IMAGE_GEOMETRY[i].to_s
      sh "mogrify -resize #{arg} #{f}"
    end
  end

  pngs.each { |f| png_optim(f) }
  jpgs.each { |f| jpg_optim(f) }

  # Optimize edilmiş resimlerin kullanıldığı slaytları, bu resimler slayta
  # gömülü olabileceğinden tekrar üretelim.  Nasıl?  Aşağıdaki berbat
  # numarayla.  Resim'e ilişik bir referans varsa dosyaya dokun.
  (pngs + jpgs).each do |f|
    name = File.basename f
    FileList["*/*.md"].each do |src|
      sh "grep -q '(.*#{name})' #{src} && touch #{src}"
    end
  end
end

# Landslide yapılandırmalarından görev bilgilerini üret
CFGFILES.each do |file|
  presentation = File.dirname(file)
  cfg          = File.basename(file)

  chdir presentation do
    config = File.open(cfg, "r") do |f|
      PythonConfig::ConfigParser.new(f)
    end

    landslide = config['landslide']
    if ! landslide
      $stderr.puts "#{file}: 'landslide' bölümü tanımlanmamış"
      exit 1
    end

    if landslide['destination']
      $stderr.puts "#{file}: 'destination' ayarı kullanılmış; hedef dosya belirtilmeyin"
      exit 1
    end

    built = presentation + '.html'
    thumbnail = File.to_herepath(presentation + '.png')
    target = File.to_herepath(built)

    # bağımlılık verilecek tüm dosyaları listele
    deps = []
    (DEPEND_ALWAYS + landslide.values_at(*DEPEND_KEYS)).compact.each do |v|
      deps += v.split.select { |p| File.exists?(p) }.map { |p| File.to_filelist(p) }.flatten
    end

    # bağımlılık ağacının çalışması için tüm yolları bu dizine göreceli yap
    deps.map! { |e| File.to_herepath(e) }
    deps.delete(target)
    deps.delete(thumbnail)

    tags = []
    order = DEFAULT_ORDER
    (config['default'] || {}).each do |k, v|
      case k
        when 'tags'
          tags += v.split
        when 'order'
          order = v
      end
    end

    PRESENTATION[presentation] = {
      :cfg       => cfg,
      :deps      => deps,
      :target    => target,
      :built     => built,
      :thumbnail => thumbnail,
      :tags      => tags,
      :order     => order,
    }
  end
end

# Etiket bilgilerini üret
PRESENTATION.each do |k, v|
  v[:tags].each do |t|
    TAG[t] ||= []
    TAG[t] << k
  end
end

# Görevleri dinamik olarak üret.  Her alt sunum dizini için bir görev
# tanımlıyoruz
tasktab = {}
PRESENTATION.each do |presentation, data|
  ns = namespace presentation do
    # Sunum dosyaları
    file data[:target] => data[:deps] do |t|
      chdir presentation do
        sh "landslide -i #{data[:cfg]}"
        # XXX: Slayt bağlamı iOS tarayıcılarında sorun çıkarıyor.  Kirli bir çözüm!
        sh 'sed -i -e "s/^\([[:blank:]]*var hiddenContext = \)false\(;[[:blank:]]*$\)/\1true\2/" presentation.html'
        mv "presentation.html", data[:built]
      end
    end

    # Küçük resimler
    file data[:thumbnail] => data[:target] do
      sh "cutycapt " +
          "--url=file://#{File.absolute_path(data[:target])}#slide1 " +
          "--out=#{data[:thumbnail]} " +
          "--user-style-string='div.slides { width: 900px; overflow: hidden; }' " +
          "--min-width=1024 " +
          "--min-height=768 " +
          "--delay=1000"
      sh "mogrify -resize 240 #{data[:thumbnail]}"
      png_optim(data[:thumbnail])
    end

    desc "resimleri optimize et"
    task :optim do
      chdir presentation do
        optim
      end
    end

    desc "inşa et"
    task :build => [:optim, data[:target], :thumbnail]

    desc "küçük resim"
    task :thumbnail => data[:thumbnail]

    desc "temizle"
    task :clean do
      rm_f data[:target]
      rm_f data[:thumbnail]
    end

    desc "öntanımlı"
    task :default => :build
  end

  # Alt görevleri görev tablosuna işle
  ns.tasks.map(&:to_s).each do |t|
    _, _, name = t.partition(":")
    tasktab[name.to_sym] ||= []
    tasktab[name.to_sym] << t
  end
end
# Görev tablosundan yararlanarak üst isim uzayında ilgili görevleri tanımla
tasktab.each { |k, v| task k => v }

# İndex için kural tanımla
file "index.html" => [*PRESENTATION.map { |k, v| v[:thumbnail] }, TEMPLATE_INDEX] do |t|
  template = ERB.new File.new(TEMPLATE_INDEX).read, nil, "%"
  File.open(t.name, "w+") do |f|
    f.write(template.result(binding))
  end
end

# İndex işlemlerini ilgili görevlere ekle
desc "İndis"
task :index   => "index.html"
task :default => :index
task :clean  do
  rm_f "index.html"
end
