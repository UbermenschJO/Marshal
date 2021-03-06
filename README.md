# Marshal

Library to marshal C style data structures. For use with legacy TCP socket communication protocols and file formats (e.g. dbf file) . Marshaling to/from clojure data structures: maps, vectors, lists, numbers, booleans, and strings using OutputStream and InputStream interfaces. Marshal big or little endian (default is little endian). 

Primitive marshaling types:
- `ubyte` - unsigned byte
- `sbyte` - signed byte
- `ushort` - unsigned 2 byte integer
- `sshort` - signed 2 byte integer
- `uint32` - unsigned 4 byte integer
- `sint32` - signed 4 byte integer
- `uint64` - unsigned 8 byte integer
- `sint64` - signed 8 byte integer
- `float` - 4 byte floating point number
- `double` - 8 byte floating point number
- `bool8` - 1 byte boolean
- `bool32` - 4 byte boolean

Functions to create composite marshalling types:
- `array` - fixed or variable length of homogenous marshaling types - marshal to/from Clojure vectors (or lists)
- `struct` - ordered keyword/marshal type pairs - marshal to/from Clojure maps
- `ascii-string` - fixed of variable length ascii string - marshal to/from Clojure strings ()
- `vector` - unamed ordered list of marshal types 

Marshaling functions (API):
- `read` - marshals a value from an InputStream 
- `write` - marshals a value to an OutputStram

## Usage

say we have a C language struct (no byte alignment padding) that has size 34 bytes (4+10+5*4) that we wish to marshal

```c
//C or C++ header file declaration
struct {
   int type;
   char name[10];
   int data[5];
};
```

```clojure
=> (require '[marshal.core :as m])
nil
;;declare a marshaling struct in the same order and using the same types as the C struct
=> (def s (m/struct :type m/sint32 :name (m/ascii-string 10) :data (m/array m/sint32 5)))
#'user/s
;;create an output stream (in lieu of an outputstream from a socket)
=> (def os (java.io.ByteArrayOutputStream.))
#'user/os
;;marshal a clojure map to the outputstream 
=> (m/write os s {:type 1 :name "1234567890" :data [1 2 3 4 5]})
34
;;create an input stream (in lieu of an inputstream from a socket)
=> (def is (clojure.java.io/input-stream (.toByteArray os)))
#'user/is
;;marshal the clojure map from the input stream 
=> (m/read is s)
{:type 1, :name "1234567890", :data [1 2 3 4 5]}
````

the following, perhaps more realistic looking example, reads a dBase file format e.g. ESRI shapefile attribute format. NOTE: it's just an example i.e. only numeric and strings are supported 

```clojure
(ns marshal.examples.dbf
  (:require [marshal.core :as m]
            [clojure.java.io]))

(def dbheaderfield (m/struct :name (m/ascii-string 11) 
                             :type m/ubyte             
                             :offset m/uint32
                             :field-size m/ubyte
                             :field-dec m/ubyte
                             :res1 (m/array  m/ubyte 2)
                             :work_area_id m/ubyte
                             :res2 (m/array m/ubyte 10)
                             :prod_index m/ubyte))

(def dbheader (m/struct :type  m/ubyte
                        :mod-year m/ubyte
                        :mod-month m/ubyte
                        :mod-day m/ubyte
                        :rec-count m/uint32
                        :header-size m/ushort
                        :rec-size m/ushort
                        :res1 (m/array m/ubyte 2)
                        :incomplete-trans m/ubyte
                        :encryption m/ubyte
                        :multi-user (m/array m/ubyte 12)
                        :prod-index m/ubyte
                        :lang-id m/ubyte
                        :res2 (m/array m/ubyte 2)
                        :fields (m/array dbheaderfield (fn [this]
                                                         (/ (- (:header-size this) 32 1)
                                                            (m/sizeof dbheaderfield))))
                        :terminator m/ubyte))

(defn read-column [s m field]
  (let [sz (:field-size field)
        name (keyword (:name field))
        val (m/read s (m/ascii-string sz))]
    (condp = (char (:type field))
      \N (assoc! m name (try
                          (Integer/parseInt val)
                          (catch java.lang.NumberFormatException e
                            (Double/parseDouble val)))) 
      \C (assoc! m name val)
      m)))

(defn read-record [s fields]
  (if (= 0x20 (.read s))
    (loop [coll (seq fields) res (transient {})]
      (if coll
        (let [[f & xs] coll]
          (recur xs (read-column s res f)))
        (persistent! res)))))
  
(defn read-records [s]
  (let [header (m/read s dbheader)
        fields (:fields header)
        sz (:rec-count header)]
    (loop [i 0 res (transient [])]
      (if (< i sz)
        (recur (inc i) (conj! res (read-record s fields)))
        (persistent! res)))))

(defn dbf [filename]
  (binding [m/*byte-order* java.nio.ByteOrder/LITTLE_ENDIAN]
    (with-open [s (clojure.java.io/input-stream filename)]
      (read-records s))))
```

## Installation

- Github: http://github.com/russellc/Marshal
- Clojars: https://clojars.org/marshal

```clojure
(defproject project-uses-marshal "1.0.0"
  :dependencies [[marshal "1.0.0"]])
```

## License

Copyright (c) 2011 Russell Christopher All rights reserved. The use and
distribution terms for this software are covered by the Eclipse Public
License 1.0 (http://opensource.org/licenses/eclipse-1.0.php) which can be found
in the file epl-v10.html at the root of this distribution. By using this
software in any fashion, you are agreeing to be bound by the terms of
this license. You must not remove this notice, or any other, from this
software.
