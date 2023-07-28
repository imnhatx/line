Sketchup.require "sketchup.rb"

module UNI
  module ONE  
    def self.explode_sub_groups_and_components(entity)
      model = Sketchup.active_model
      entity.entities.each do |e|
        if e.is_a?(Sketchup::Group) || e.is_a?(Sketchup::ComponentInstance)
          model.start_operation('explode', true)
          explode_sub_groups_and_components(e)
          e.explode
          model.commit_operation
        end
      end
    end

    def self.is_edge_group?(entity)
      return false unless entity.is_a?(Sketchup::Group)
      entity.entities.all? { |e| e.is_a?(Sketchup::Edge) }
    end

    def self.make_group_for_each_face(group_or_component)
      model = Sketchup.active_model
      entities = group_or_component.definition.entities

      # Check if the group is Cuboid or Edge group, if so, skip processing
      if is_cuboid_group_with_3_dimensions?(group_or_component) || is_edge_group?(group_or_component)
        return
      end

      model.start_operation('Group Faces', true)
      faces = entities.grep(Sketchup::Face)
      new_groups = []
      faces.each do |face|
        new_group = entities.add_group
        new_group.entities.add_face(face.vertices.map(&:position))
        new_groups << new_group
      end
      model.commit_operation

      transformation = Geom::Transformation.new
      group_or_component.transformation.invert!
      transformation *= group_or_component.transformation

      model.start_operation('Copy to Model Space', true)
      copied_groups = []
      new_groups.each do |new_group|
        copied_group = model.entities.add_instance(new_group.definition, transformation)
        copied_groups << copied_group
      end
      model.commit_operation

      model.start_operation('Erase Original Groups', true)
      group_or_component.erase!
      model.commit_operation

      # Filter selected entities to include only Groups and ComponentInstances with 1 Face or 1 Edge.
      selected_entities = model.selection.select do |e|
        (e.is_a?(Sketchup::Group) || e.is_a?(Sketchup::ComponentInstance)) &&
          (e.entities.grep(Sketchup::Face).count >= 1 || e.entities.grep(Sketchup::Edge).count >= 1)
      end

      model.selection.clear
      selected_entities.each { |e| model.selection.add(e) }
      copied_groups.each { |copied_group| model.selection.add(copied_group) }
    end
    
    # Lấy model hiện tại
    model = Sketchup.active_model
    
    # Lấy ra tất cả các phần tử được chọn
    selection = model.selection
    
    # Kiểm tra xem có phần tử nào được chọn không
    if selection.empty?
      #UI.messagebox('Vui lòng chọn một nhóm hoặc component.')
    else
      # Duyệt qua tất cả các đối tượng trong selection
      selection.each do |group_or_component|
        # Kiểm tra xem phần tử đó có phải là một Group hoặc ComponentInstance không
        if group_or_component.is_a?(Sketchup::Group) || group_or_component.is_a?(Sketchup::ComponentInstance)
          make_group_for_each_face(group_or_component)
        else
          #UI.messagebox('Vui lòng chỉ chọn các nhóm hoặc components.')
        end
      end
    end
    
    model = Sketchup.active_model
    selection = model.selection
    
    def model.find_all_connected(edges)
      edges1 = edges.clone
      list = []
    
      while edges1.length > 0
        connected_edges = []
        edge = edges1.find { |e| e.vertices[0].degree == 1 || e.vertices[1].degree == 1 }
        edge ||= edges1.shift
    
        connected_edges << edge
    
        while (connected_edge = connected_edges.last.connected_edges & edges1).any?
          connected_edges << connected_edge[0]
        end
    
        list << connected_edges
        edges1 -= connected_edges
      end
    
      list
    rescue => error
      puts error, error.backtrace.join('\n')
      edges
    end
    
    
    def model.weld(mode)
      @limit_edges = 2000
      model = Sketchup.active_model
      entities = model.active_entities
      edges = model.selection.grep(Sketchup::Edge)
      edges_face = model.selection.grep(Sketchup::Face).flat_map { |f| f.loops.flat_map(&:edges) }
      edges = edges_face + (edges - edges_face)
      edges = edges.reject { |e| e.hidden? || e.soft? }.uniq
    
      model.start_operation('Weld', true, false, false)
      curs = 0
      if Sketchup.version.to_f < 20.1
        Sketchup.status_text = "Processing Find Edges"
        edge_list_connected = find_all_connected(edges)
        ver_list_connected = edge_list_connected.flat_map { |eds| find_vertices(eds) }
        ver_list = ver_list_connected.flatten(1)
    
        ver_list_connected.each do |eds|
          eds.each do |ed|
            if ed.length > 1
              points = ed
              entities.add_curve(points)
              curs += 1
            end
            Sketchup.status_text = "Processing Curve: #{curs}/#{ver_list.length}"
          end
        end
    
        model.selection.clear
        if mode == 0
          count = []
          ver_list.each do |ed|
            eds = ed.flat_map { |e| e.edges } & edges
            eds.each { |e| e.explode_curve if e.valid? && e.curve }
          end
        end
      else
        model.selection.clear
        curves = model.active_entities.weld(edges)
        model.selection.add(curves.flat_map(&:edges))
        curs = curves.length
      end
    
      #puts "Weld #{curs} curves"
      model.commit_operation
    rescue => error
      puts error, error.backtrace.join('\n')
      model.abort_operation if model
    end
    


    def self.set_group_name(group, name)
      group.name = name
    end 

    def self.find_or_create_layer(layer_name)
      model = Sketchup.active_model
      layer = model.layers[layer_name]
  
      unless layer
        layer = model.layers.add(layer_name)
      end
      layer
    end

    def self.set_entities_layer(entities, layer_name)
      layer = find_or_create_layer(layer_name)
      entities.each do |entity|
        entity.layer = layer
      end
    end

    def self.create_group_with_name(name, selected_entities)
      model = Sketchup.active_model
      entities = model.active_entities
  
      group = entities.add_group(selected_entities)
      group.name = name
  
      group
    end

    def self.valid_color_code?(color_code)
      # Hàm kiểm tra xem người dùng đã nhập mã màu HEX hay không
      # Nếu chuỗi có 6 ký tự và là một mã màu HEX hợp lệ thì trả về true
      return false unless color_code.is_a?(String)
      color_code.match?(/\A\b([0-9A-Fa-f]{6})\b\z/)
    end

    def self.get_color_code(color_name)
      # Hàm chuyển đổi tên màu thành mã màu tương ứng
      return color_name.upcase if valid_color_code?(color_name) # Nếu đã nhập mã màu HEX thì trả về luôn
    
      # Kiểm tra nếu người dùng nhập các tên màu có hoặc không dấu
      case color_name.downcase.gsub(/[\s_]/, '') # Xóa khoảng trắng và dấu gạch dưới trong chuỗi đầu vào
      when 'do', 'đỏ'
        'CD5C5C' # Red
      when 'xanh la', 'xanh lá'
        '2E8B57' # Green
      when 'vang', 'vàng'
        'FFCC99' # Yellow
      when 'cam'
        'FFA500' # Orange
      when 'tim', 'tím'
        '800080' # Purple
      when 'cham', 'chàm'
        '8B658B'
      when 'trang', 'trắng'
        'FFFFFF' # White
      when 'nau', 'nâu'
        '8B4513'
      when 'xanh ngoc', 'xanh ngọc'
        'B0E0E6'
      when 'xam', 'xám'
        'C0C0C0'
      when 'tia', 'tía'
        '5D478B'
      when 'vang hong', 'vàng hồng'
        'FFE4E1'
      when 'xanh', 'xanh'
        '6495ED'
      when 'hong', 'hồng'
        'DB7093'
      when 'huong', 'hường'
        'FFC1C1'
      when 'den', 'đen'
        '000000' # Black
      else
        '778996' # Mặc định là màu xanh nếu không nhận dạng được tên màu
      end
    end
    

    def self.add_to_cuboid_group_safe(entity, cuboid_group)
      # Save the transformations of the entity and cuboid group
      entity_transformation = entity.transformation
      cuboid_transformation = cuboid_group.transformation
    
      # Compute the transformation needed to move the entity into the cuboid group
      transformation_to_cuboid = cuboid_transformation.inverse * entity_transformation
    
      # Create a new instance of the entity and add it to the cuboid group
      new_instance = cuboid_group.entities.add_instance(entity.definition, transformation_to_cuboid)
    
      # Set name, material, and layer for the new instance
      new_instance.name = entity.name
      new_instance.material = entity.material
      new_instance.layer = entity.layer
    
      # Erase the original entity
      entity.erase!
    end

    def self.is_cuboid_group_with_3_dimensions?(entity)
      # Hàm kiểm tra xem entity có phải là cuboid group có đủ 3 chiều dài, cao, rộng hay không
      return false unless entity.is_a?(Sketchup::Group)
      edges_count = entity.entities.grep(Sketchup::Edge).count
      faces_count = entity.entities.grep(Sketchup::Face).count
      edges_count == 12 && faces_count == 6
    end

    def self.show_name_layer_inputbox
      model = Sketchup.active_model
      selection = model.selection
    
      show_message_box = true # Biến cờ để kiểm tra xem có hiển thị messagebox hay không
    

      # Explode các group lồng bên trong trước khi xử lý
      selection.each do |entity|
        if entity.is_a?(Sketchup::Group)
          unless is_cuboid_group_with_3_dimensions?(entity)
            explode_sub_groups_and_components(entity)
          end
        else
          UI.messagebox("HÃY MAKE GROUP ĐỐI TƯỢNG.")
          return
        end
      end
      
      selection.each do |entity|
        #unless is_cuboid_group_with_3_dimensions?(entity)
        make_group_for_each_face(entity) unless entity.entities.grep(Sketchup::Face).count == 1
        #end
      end
      
      # Sau khi thực hiện xong, kiểm tra biến cờ và hiển thị thông báo
      if show_message_box
        # UI.messagebox("LỖI ĐÃ ĐƯỢC SỬA")
      end
    
      # Filter selected entities to include only Groups and ComponentInstances with 1 Face or 1 Edge.
      selected_entities = selection.select do |e|
        (e.is_a?(Sketchup::Group) || e.is_a?(Sketchup::ComponentInstance)) &&
          (e.entities.grep(Sketchup::Face).count >= 1 || e.entities.grep(Sketchup::Edge).count >= 1)
      end
    
      if selected_entities.empty?
        UI.messagebox("CHỌN ĐỐI TƯỢNG TẠO ĐƯỜNG TUỲ BIẾN.")
        return
      end

      selected_entities.each do |entity|
        unless entity.is_a?(Sketchup::Group)
          UI.messagebox("HÃY MAKE GROUP CHO ĐỐI TƯỢNG ĐƯỢC CHỌN.")
          return
        end
      end

      # Start a new operation (for undo functionality)
      model.start_operation('Weld Groups', true)

      selection.to_a.each do |selected|
        unless is_cuboid_group_with_3_dimensions?(selected)
          # Check if the selected item is a group
          if selected.is_a?(Sketchup::Group)
            # Check if the group contains any faces
            next if selected.entities.any? { |e| e.is_a?(Sketchup::Face) }

            # Open the selected group (this sets it as the active entities)
            model.active_path = [selected]

            # Clear the current selection
            model.selection.clear

            # Select all entities in the group
            selected.entities.each { |e| model.selection.add(e) }

            # Call weld method on the selected entities
            model.weld(0)

            # Close the group (this sets active entities back to model entities)
            model.close_active
          end
        end
      end

      # Commit the operation
      model.commit_operation

      prompts = ['TÊN LAYER   : ', 'MÀU SẮC      : ']
      default_values = ['', 'UNI']
      list_of_colors = ['Red', 'Green', 'Blue', 'Yellow', 'Orange', 'Purple', 'White', 'Black'] # Thêm danh sách màu hỗ trợ
      results = UI.inputbox(prompts, default_values, 'TẠO ĐƯỜNG TUỲ BIẾN')
    
      if results
        edge_group_name = "ABF_" # Thêm dòng này để nhận tên cho edge_group
        face_group_name = "ABF_" # Thêm dòng này để nhận tên cho edge_group
        layer_name = results[0]
        layer_name = 'Untagged' if layer_name.empty?
        edge_group_name = "ABF_#{edge_group_name}" if !edge_group_name.start_with?('ABF_') # Đặt tiền tố cho tên edge_group
        face_group_name = "ABF_#{edge_group_name}" if !edge_group_name.start_with?('ABF_') # Đặt tiền tố cho tên face_group
        face_group_color = get_color_code(results[1]) # Chuyển đổi tên màu thành mã màu tương ứng
    
        model.start_operation('Move Group', true)
    
        cuboid_group = selected_entities.find { |e| is_cuboid_group_with_3_dimensions?(e) }
    
        selected_entities.each do |entity|
          if entity.is_a?(Sketchup::Group) && entity.entities.grep(Sketchup::Face).count == 1
            # Process face group
            face_group = entity
            face_group_material = face_group.material
            face_group_layer = find_or_create_layer(layer_name) # Find or create the layer for face group
            set_group_name(face_group, face_group_name)
            set_entities_layer([face_group] + face_group.entities.to_a, layer_name)
    
            face_group_material = Sketchup::Color.new(face_group_color)
            face_group.material = face_group_material
    
            add_to_cuboid_group_safe(face_group, cuboid_group) if cuboid_group
          elsif entity.is_a?(Sketchup::Group) && entity.entities.all? { |e| e.is_a?(Sketchup::Edge) }
            # Process edge group
            edge_group = entity
            edge_group_material = edge_group.material
            edge_group_layer = find_or_create_layer(layer_name)
            set_group_name(edge_group, edge_group_name)
            set_entities_layer([edge_group] + edge_group.entities.to_a, layer_name)
    
            add_to_cuboid_group_safe(edge_group, cuboid_group) if cuboid_group
          end
        end
        model.commit_operation
      end
    end
  end
end
