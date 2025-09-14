
    -- 1) CREACIÓN DE TABLAS
 
    DROP TABLE libros CASCADE CONSTRAINTS;
    DROP TABLE autores CASCADE CONSTRAINTS;
    DROP TABLE categorias CASCADE CONSTRAINTS;
    
    -- Autores
    CREATE TABLE autores (
      id_autor NUMBER PRIMARY KEY,
      nombre VARCHAR2(200) NOT NULL
    );
    
    -- Categorías
    CREATE TABLE categorias (
      id_categoria NUMBER PRIMARY KEY,
      nombre VARCHAR2(100) NOT NULL
    );
    
    -- Libros
    CREATE TABLE libros (
      id_libro NUMBER PRIMARY KEY,
      titulo VARCHAR2(200) NOT NULL,
      id_autor NUMBER REFERENCES autores(id_autor),
      id_categoria NUMBER REFERENCES categorias(id_categoria),
      precio NUMBER(10,2) CHECK(precio >= 0),
      stock NUMBER DEFAULT 0 CHECK(stock >= 0)
    );
    

    -- 2) INSERTAR DATOS DE PRUEBA

    -- Autores
    INSERT INTO autores(id_autor, nombre) VALUES (1, 'Gabriel García Márquez');
    INSERT INTO autores(id_autor, nombre) VALUES (2, 'Isabel Allende');
    INSERT INTO autores(id_autor, nombre) VALUES (3, 'George Orwell');
    
    -- Categorías
    INSERT INTO categorias(id_categoria, nombre) VALUES (1, 'Novela');
    INSERT INTO categorias(id_categoria, nombre) VALUES (2, 'Ficción histórica');
    INSERT INTO categorias(id_categoria, nombre) VALUES (3, 'Ciencia ficción');
    
    -- Libros
    INSERT INTO libros(id_libro, titulo, id_autor, id_categoria, precio, stock)
    VALUES (1, 'Cien Años de Soledad', 1, 1, 12000, 5);
    
    INSERT INTO libros(id_libro, titulo, id_autor, id_categoria, precio, stock)
    VALUES (2, 'La Casa de los Espíritus', 2, 2, 9000, 3);
    
    INSERT INTO libros(id_libro, titulo, id_autor, id_categoria, precio, stock)
    VALUES (3, '1984', 3, 3, 8000, 10);
    
    COMMIT;
    
    
    -- 3) BLOQUE 1: RECORD
    
    DECLARE
      TYPE registro_libro IS RECORD (
        titulo    libros.titulo%TYPE,
        autor     autores.nombre%TYPE,
        stock     libros.stock%TYPE,
        categoria categorias.nombre%TYPE
      );
      v_libro registro_libro;
    BEGIN
      SELECT l.titulo, a.nombre, l.stock, c.nombre
      INTO v_libro
      FROM libros l
      JOIN autores a ON l.id_autor = a.id_autor
      JOIN categorias c ON l.id_categoria = c.id_categoria
      WHERE l.id_libro = 1;
    
      DBMS_OUTPUT.PUT_LINE('Título: '   || v_libro.titulo);
      DBMS_OUTPUT.PUT_LINE('Autor: '    || v_libro.autor);
      DBMS_OUTPUT.PUT_LINE('Categoría: '|| v_libro.categoria);
      DBMS_OUTPUT.PUT_LINE('Stock: '    || v_libro.stock);
    END;
    /
    
 
    -- 4) BLOQUE 2: VARRAY

    DECLARE
      TYPE top5_libros IS VARRAY(5) OF VARCHAR2(200);
      v_top5 top5_libros := top5_libros();
    BEGIN
      v_top5.EXTEND(5);
      v_top5(1) := 'Cien Años de Soledad';
      v_top5(2) := 'La Casa de los Espíritus';
      v_top5(3) := '1984';
      v_top5(4) := 'Rayuela';
      v_top5(5) := 'Don Quijote de la Mancha';
    
      FOR i IN 1..v_top5.COUNT LOOP
        DBMS_OUTPUT.PUT_LINE('Top ' || i || ': ' || v_top5(i));
      END LOOP;
    END;
    /
    
  
    -- 5) BLOQUE 3: CURSOR CON PARÁMETRO (actualizado)
   
    DECLARE
      CURSOR c_libros_por_autor (p_id_autor NUMBER) IS
        SELECT 
          l.id_libro,
          l.titulo,
          l.precio,
          l.stock,
          a.nombre AS nombre_autor,
          c.nombre AS categoria
        FROM libros l
        JOIN autores a ON l.id_autor = a.id_autor
        LEFT JOIN categorias c ON l.id_categoria = c.id_categoria
        WHERE l.id_autor = p_id_autor;
    
      v_libro_record c_libros_por_autor%ROWTYPE;
      v_id_autor NUMBER := 1; -- Cambiar entre 1,2,3
    BEGIN
      OPEN c_libros_por_autor(v_id_autor);
    
      DBMS_OUTPUT.PUT_LINE('--- Libros del autor ID: ' || v_id_autor || ' ---');
    
      LOOP
        FETCH c_libros_por_autor INTO v_libro_record;
        EXIT WHEN c_libros_por_autor%NOTFOUND;
    
        DBMS_OUTPUT.PUT_LINE('Libro ID: ' || v_libro_record.id_libro);
        DBMS_OUTPUT.PUT_LINE(' - Título: ' || v_libro_record.titulo);
        DBMS_OUTPUT.PUT_LINE(' - Autor: ' || v_libro_record.nombre_autor);
        DBMS_OUTPUT.PUT_LINE(' - Categoría: ' || v_libro_record.categoria);
        DBMS_OUTPUT.PUT_LINE(' - Precio: ' || v_libro_record.precio);
        DBMS_OUTPUT.PUT_LINE(' - Stock: '  || v_libro_record.stock);
        DBMS_OUTPUT.PUT_LINE('----------------------------------------');
      END LOOP;
    
      CLOSE c_libros_por_autor;
    END;
    /
    
  
    -- 6) BLOQUE 4: EXCEPCIONES
  
    DECLARE
      ex_stock_negativo EXCEPTION;
      ex_precio_invalido EXCEPTION;
    
      v_stock NUMBER := -5;
      v_precio NUMBER := -100;
    BEGIN
      IF v_stock < 0 THEN
        RAISE ex_stock_negativo;
      END IF;
    
      IF v_precio < 0 THEN
        RAISE ex_precio_invalido;
      END IF;
    
      DBMS_OUTPUT.PUT_LINE('Datos válidos: Stock=' || v_stock || ', Precio=' || v_precio);
    
    EXCEPTION
      WHEN ex_stock_negativo THEN
        DBMS_OUTPUT.PUT_LINE('️ Error: El stock no puede ser negativo (' || v_stock || ')');
      WHEN ex_precio_invalido THEN
        DBMS_OUTPUT.PUT_LINE('️ Error: El precio no puede ser negativo (' || v_precio || ')');
      WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('️ Error inesperado: ' || SQLERRM);
    END;
    /
    
   
    -- 7) TRIGGER: AUDITAR CAMBIOS EN PRECIO Y STOCK

    CREATE OR REPLACE TRIGGER trg_auditoria_libros
    BEFORE UPDATE ON libros
    FOR EACH ROW
    BEGIN
      IF :OLD.precio <> :NEW.precio OR :OLD.stock <> :NEW.stock THEN
        DBMS_OUTPUT.PUT_LINE(' Cambio detectado en libro ID ' || :OLD.id_libro ||
                             ': Precio ' || :OLD.precio || ' → ' || :NEW.precio ||
                             ', Stock ' || :OLD.stock || ' → ' || :NEW.stock);
      END IF;
    END;
    /
    
  
    -- 8) SCRIPT PARA VALIDAR EL TRIGGER
 
    UPDATE libros
    SET precio = 15000, stock = 7
    WHERE id_libro = 1;
    
    UPDATE libros
    SET precio = 9500
    WHERE id_libro = 2;
    
    UPDATE libros
    SET stock = 12
    WHERE id_libro = 3;
    
    COMMIT;
