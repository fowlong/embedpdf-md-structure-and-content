# Part 5


  fsync(tty) {
          if (tty.output && tty.output.length > 0) {
            err(UTF8ArrayToString(tty.output));
            tty.output = [];
          }
        },
  },
  };
  
  
  var zeroMemory = (address, size) => {
      HEAPU8.fill(0, address, address + size);
    };
  
  var alignMemory = (size, alignment) => {
      assert(alignment, "alignment argument is required");
      return Math.ceil(size / alignment) * alignment;
    };
  var mmapAlloc = (size) => {
      size = alignMemory(size, 65536);
      var ptr = _emscripten_builtin_memalign(65536, size);
      if (ptr) zeroMemory(ptr, size);
      return ptr;
    };
  var MEMFS = {
  ops_table:null,
  mount(mount) {
        return MEMFS.createNode(null, '/', 16384 | 511 /* 0777 */, 0);
      },
  createNode(parent, name, mode, dev) {
        if (FS.isBlkdev(mode) || FS.isFIFO(mode)) {
          // no supported
          throw new FS.ErrnoError(63);
        }
        MEMFS.ops_table ||= {
          dir: {
            node: {
              getattr: MEMFS.node_ops.getattr,
              setattr: MEMFS.node_ops.setattr,
              lookup: MEMFS.node_ops.lookup,
              mknod: MEMFS.node_ops.mknod,
              rename: MEMFS.node_ops.rename,
              unlink: MEMFS.node_ops.unlink,
              rmdir: MEMFS.node_ops.rmdir,
              readdir: MEMFS.node_ops.readdir,
              symlink: MEMFS.node_ops.symlink
            },
            stream: {
              llseek: MEMFS.stream_ops.llseek
            }
          },
          file: {
            node: {
              getattr: MEMFS.node_ops.getattr,
              setattr: MEMFS.node_ops.setattr
            },
            stream: {
              llseek: MEMFS.stream_ops.llseek,
              read: MEMFS.stream_ops.read,
              write: MEMFS.stream_ops.write,
              allocate: MEMFS.stream_ops.allocate,
              mmap: MEMFS.stream_ops.mmap,
              msync: MEMFS.stream_ops.msync
            }
          },
          link: {
            node: {
              getattr: MEMFS.node_ops.getattr,
              setattr: MEMFS.node_ops.setattr,
              readlink: MEMFS.node_ops.readlink
            },
            stream: {}
          },
          chrdev: {
            node: {
              getattr: MEMFS.node_ops.getattr,
              setattr: MEMFS.node_ops.setattr
            },
            stream: FS.chrdev_stream_ops
          }
        };
        var node = FS.createNode(parent, name, mode, dev);
        if (FS.isDir(node.mode)) {
          node.node_ops = MEMFS.ops_table.dir.node;
          node.stream_ops = MEMFS.ops_table.dir.stream;
          node.contents = {};
        } else if (FS.isFile(node.mode)) {
          node.node_ops = MEMFS.ops_table.file.node;
          node.stream_ops = MEMFS.ops_table.file.stream;
          node.usedBytes = 0; // The actual number of bytes used in the typed array, as opposed to contents.length which gives the whole capacity.
          // When the byte data of the file is populated, this will point to either a typed array, or a normal JS array. Typed arrays are preferred
          // for performance, and used by default. However, typed arrays are not resizable like normal JS arrays are, so there is a small disk size
          // penalty involved for appending file writes that continuously grow a file similar to std::vector capacity vs used -scheme.
          node.contents = null; 
        } else if (FS.isLink(node.mode)) {
          node.node_ops = MEMFS.ops_table.link.node;
          node.stream_ops = MEMFS.ops_table.link.stream;
        } else if (FS.isChrdev(node.mode)) {
          node.node_ops = MEMFS.ops_table.chrdev.node;
          node.stream_ops = MEMFS.ops_table.chrdev.stream;
        }
        node.timestamp = Date.now();
        // add the new node to the parent
        if (parent) {
          parent.contents[name] = node;
          parent.timestamp = node.timestamp;
        }
        return node;
      },
  getFileDataAsTypedArray(node) {
        if (!node.contents) return new Uint8Array(0);
        if (node.contents.subarray) return node.contents.subarray(0, node.usedBytes); // Make sure to not return excess unused bytes.
        return new Uint8Array(node.contents);
      },
  expandFileStorage(node, newCapacity) {
        var prevCapacity = node.contents ? node.contents.length : 0;
        if (prevCapacity >= newCapacity) return; // No need to expand, the storage was already large enough.
        // Don't expand strictly to the given requested limit if it's only a very small increase, but instead geometrically grow capacity.
        // For small filesizes (<1MB), perform size*2 geometric increase, but for large sizes, do a much more conservative size*1.125 increase to
        // avoid overshooting the allocation cap by a very large margin.
        var CAPACITY_DOUBLING_MAX = 1024 * 1024;
        newCapacity = Math.max(newCapacity, (prevCapacity * (prevCapacity < CAPACITY_DOUBLING_MAX ? 2.0 : 1.125)) >>> 0);
        if (prevCapacity != 0) newCapacity = Math.max(newCapacity, 256); // At minimum allocate 256b for each file when expanding.
        var oldContents = node.contents;
        node.contents = new Uint8Array(newCapacity); // Allocate new storage.
        if (node.usedBytes > 0) node.contents.set(oldContents.subarray(0, node.usedBytes), 0); // Copy old data over to the new storage.
      },
  resizeFileStorage(node, newSize) {
        if (node.usedBytes == newSize) return;
        if (newSize == 0) {
          node.contents = null; // Fully decommit when requesting a resize to zero.
          node.usedBytes = 0;
        } else {
          var oldContents = node.contents;
          node.contents = new Uint8Array(newSize); // Allocate new storage.
          if (oldContents) {
            node.contents.set(oldContents.subarray(0, Math.min(newSize, node.usedBytes))); // Copy old data over to the new storage.
          }
          node.usedBytes = newSize;
        }
      },
  node_ops:{
  getattr(node) {
          var attr = {};
          // device numbers reuse inode numbers.
          attr.dev = FS.isChrdev(node.mode) ? node.id : 1;
          attr.ino = node.id;
          attr.mode = node.mode;
          attr.nlink = 1;
          attr.uid = 0;
          attr.gid = 0;
          attr.rdev = node.rdev;
          if (FS.isDir(node.mode)) {
            attr.size = 4096;
          } else if (FS.isFile(node.mode)) {
            attr.size = node.usedBytes;
          } else if (FS.isLink(node.mode)) {
            attr.size = node.link.length;
          } else {
            attr.size = 0;
          }
          attr.atime = new Date(node.timestamp);
          attr.mtime = new Date(node.timestamp);
          attr.ctime = new Date(node.timestamp);
          // NOTE: In our implementation, st_blocks = Math.ceil(st_size/st_blksize),
          //       but this is not required by the standard.
          attr.blksize = 4096;
          attr.blocks = Math.ceil(attr.size / attr.blksize);
          return attr;
        },
  setattr(node, attr) {
          if (attr.mode !== undefined) {
            node.mode = attr.mode;
          }
          if (attr.timestamp !== undefined) {
            node.timestamp = attr.timestamp;
          }
          if (attr.size !== undefined) {
            MEMFS.resizeFileStorage(node, attr.size);
          }
        },
  lookup(parent, name) {
          throw FS.genericErrors[44];
        },
  mknod(parent, name, mode, dev) {
          return MEMFS.createNode(parent, name, mode, dev);
        },
  rename(old_node, new_dir, new_name) {
          // if we're overwriting a directory at new_name, make sure it's empty.
          if (FS.isDir(old_node.mode)) {
            var new_node;
            try {
              new_node = FS.lookupNode(new_dir, new_name);
            } catch (e) {
            }
            if (new_node) {
              for (var i in new_node.contents) {
                throw new FS.ErrnoError(55);
              }
            }
          }
          // do the internal rewiring
          delete old_node.parent.contents[old_node.name];
          old_node.parent.timestamp = Date.now()
          old_node.name = new_name;
          new_dir.contents[new_name] = old_node;
          new_dir.timestamp = old_node.parent.timestamp;
        },
  unlink(parent, name) {
          delete parent.contents[name];
          parent.timestamp = Date.now();
        },
  rmdir(parent, name) {
          var node = FS.lookupNode(parent, name);
          for (var i in node.contents) {
            throw new FS.ErrnoError(55);
          }
          delete parent.contents[name];
          parent.timestamp = Date.now();
        },
  readdir(node) {
          var entries = ['.', '..'];
          for (var key of Object.keys(node.contents)) {
            entries.push(key);
          }
          return entries;
        },
  symlink(parent, newname, oldpath) {
          var node = MEMFS.createNode(parent, newname, 511 /* 0777 */ | 40960, 0);
          node.link = oldpath;
          return node;
        },
  readlink(node) {
          if (!FS.isLink(node.mode)) {
            throw new FS.ErrnoError(28);
          }
          return node.link;
        },
  },
  stream_ops:{
  read(stream, buffer, offset, length, position) {
          var contents = stream.node.contents;
          if (position >= stream.node.usedBytes) return 0;
          var size = Math.min(stream.node.usedBytes - position, length);
          assert(size >= 0);
          if (size > 8 && contents.subarray) { // non-trivial, and typed array
            buffer.set(contents.subarray(position, position + size), offset);
          } else {
            for (var i = 0; i < size; i++) buffer[offset + i] = contents[position + i];
          }
          return size;
        },
  write(stream, buffer, offset, length, position, canOwn) {
          // The data buffer should be a typed array view
          assert(!(buffer instanceof ArrayBuffer));
          // If the buffer is located in main memory (HEAP), and if
          // memory can grow, we can't hold on to references of the
          // memory buffer, as they may get invalidated. That means we
          // need to do copy its contents.
          if (buffer.buffer === HEAP8.buffer) {
            canOwn = false;
          }
  
          if (!length) return 0;
          var node = stream.node;
          node.timestamp = Date.now();
  
          if (buffer.subarray && (!node.contents || node.contents.subarray)) { // This write is from a typed array to a typed array?
            if (canOwn) {
              assert(position === 0, 'canOwn must imply no weird position inside the file');
              node.contents = buffer.subarray(offset, offset + length);
              node.usedBytes = length;
              return length;
            } else if (node.usedBytes === 0 && position === 0) { // If this is a simple first write to an empty file, do a fast set since we don't need to care about old data.
              node.contents = buffer.slice(offset, offset + length);
              node.usedBytes = length;
              return length;
            } else if (position + length <= node.usedBytes) { // Writing to an already allocated and used subrange of the file?
              node.contents.set(buffer.subarray(offset, offset + length), position);
              return length;
            }
          }
  
          // Appending to an existing file and we need to reallocate, or source data did not come as a typed array.
          MEMFS.expandFileStorage(node, position+length);
          if (node.contents.subarray && buffer.subarray) {
            // Use typed array write which is available.
            node.contents.set(buffer.subarray(offset, offset + length), position);
          } else {
            for (var i = 0; i < length; i++) {
             node.contents[position + i] = buffer[offset + i]; // Or fall back to manual write if not.
            }
          }
          node.usedBytes = Math.max(node.usedBytes, position + length);
          return length;
        },
  llseek(stream, offset, whence) {
          var position = offset;
          if (whence === 1) {
            position += stream.position;
          } else if (whence === 2) {
            if (FS.isFile(stream.node.mode)) {
              position += stream.node.usedBytes;
            }
          }
          if (position < 0) {
            throw new FS.ErrnoError(28);
          }
          return position;
        },
  allocate(stream, offset, length) {
          MEMFS.expandFileStorage(stream.node, offset + length);
          stream.node.usedBytes = Math.max(stream.node.usedBytes, offset + length);
        },
  mmap(stream, length, position, prot, flags) {
          if (!FS.isFile(stream.node.mode)) {
            throw new FS.ErrnoError(43);
          }
          var ptr;
          var allocated;
          var contents = stream.node.contents;
          // Only make a new copy when MAP_PRIVATE is specified.
          if (!(flags & 2) && contents && contents.buffer === HEAP8.buffer) {
            // We can't emulate MAP_SHARED when the file is not backed by the
            // buffer we're mapping to (e.g. the HEAP buffer).
            allocated = false;
            ptr = contents.byteOffset;
          } else {
            allocated = true;
            ptr = mmapAlloc(length);
            if (!ptr) {
              throw new FS.ErrnoError(48);
            }
            if (contents) {
              // Try to avoid unnecessary slices.
              if (position > 0 || position + length < contents.length) {
                if (contents.subarray) {
                  contents = contents.subarray(position, position + length);
                } else {
                  contents = Array.prototype.slice.call(contents, position, position + length);
                }
              }
              HEAP8.set(contents, ptr);
            }
          }
          return { ptr, allocated };
        },
  msync(stream, buffer, offset, length, mmapFlags) {
          MEMFS.stream_ops.write(stream, buffer, 0, length, offset, false);
          // should we check if bytesWritten and length are the same?
          return 0;
        },
  },
  };
  
  /** @param {boolean=} noRunDep */
  var asyncLoad = (url, onload, onerror, noRunDep) => {
      var dep = !noRunDep ? getUniqueRunDependency(`al ${url}`) : '';
      readAsync(url).then(
        (arrayBuffer) => {
          assert(arrayBuffer, `Loading data file "${url}" failed (no arrayBuffer).`);
          onload(new Uint8Array(arrayBuffer));
          if (dep) removeRunDependency(dep);
        },
        (err) => {
          if (onerror) {
            onerror();
          } else {
            throw `Loading data file "${url}" failed.`;
          }
        }
      );
      if (dep) addRunDependency(dep);
    };
  
  
  var FS_createDataFile = (parent, name, fileData, canRead, canWrite, canOwn) => {
      FS.createDataFile(parent, name, fileData, canRead, canWrite, canOwn);
    };
  
  var preloadPlugins = Module['preloadPlugins'] || [];
  var FS_handledByPreloadPlugin = (byteArray, fullname, finish, onerror) => {
      // Ensure plugins are ready.
      if (typeof Browser != 'undefined') Browser.init();
  
      var handled = false;
      preloadPlugins.forEach((plugin) => {
        if (handled) return;
        if (plugin['canHandle'](fullname)) {
          plugin['handle'](byteArray, fullname, finish, onerror);
          handled = true;
        }
      });
      return handled;
    };
  var FS_createPreloadedFile = (parent, name, url, canRead, canWrite, onload, onerror, dontCreateFile, canOwn, preFinish) => {
      // TODO we should allow people to just pass in a complete filename instead
      // of parent and name being that we just join them anyways
      var fullname = name ? PATH_FS.resolve(PATH.join2(parent, name)) : parent;
      var dep = getUniqueRunDependency(`cp ${fullname}`); // might have several active requests for the same fullname
      function processData(byteArray) {
        function finish(byteArray) {
          preFinish?.();
          if (!dontCreateFile) {
            FS_createDataFile(parent, name, byteArray, canRead, canWrite, canOwn);
          }
          onload?.();
          removeRunDependency(dep);
        }
        if (FS_handledByPreloadPlugin(byteArray, fullname, finish, () => {
          onerror?.();
          removeRunDependency(dep);
        })) {
          return;
        }
        finish(byteArray);
      }
      addRunDependency(dep);
      if (typeof url == 'string') {
        asyncLoad(url, processData, onerror);
      } else {
        processData(url);
      }
    };
  
  var FS_modeStringToFlags = (str) => {
      var flagModes = {
        'r': 0,
        'r+': 2,
        'w': 512 | 64 | 1,
        'w+': 512 | 64 | 2,
        'a': 1024 | 64 | 1,
        'a+': 1024 | 64 | 2,
      };
      var flags = flagModes[str];
      if (typeof flags == 'undefined') {
        throw new Error(`Unknown file open mode: ${str}`);
      }
      return flags;
    };
  
  var FS_getMode = (canRead, canWrite) => {
      var mode = 0;
      if (canRead) mode |= 292 | 73;
      if (canWrite) mode |= 146;
      return mode;
    };
  
  
  
  
  
  
  var strError = (errno) => {
      return UTF8ToString(_strerror(errno));
    };
  
  var ERRNO_CODES = {
      'EPERM': 63,
      'ENOENT': 44,
      'ESRCH': 71,
      'EINTR': 27,
      'EIO': 29,
      'ENXIO': 60,
      'E2BIG': 1,
      'ENOEXEC': 45,
      'EBADF': 8,
      'ECHILD': 12,
      'EAGAIN': 6,
      'EWOULDBLOCK': 6,
      'ENOMEM': 48,
      'EACCES': 2,
      'EFAULT': 21,
      'ENOTBLK': 105,
      'EBUSY': 10,
      'EEXIST': 20,
      'EXDEV': 75,
      'ENODEV': 43,
      'ENOTDIR': 54,
      'EISDIR': 31,
      'EINVAL': 28,
      'ENFILE': 41,
      'EMFILE': 33,
      'ENOTTY': 59,
      'ETXTBSY': 74,
      'EFBIG': 22,
      'ENOSPC': 51,
      'ESPIPE': 70,
      'EROFS': 69,
      'EMLINK': 34,
      'EPIPE': 64,
      'EDOM': 18,
      'ERANGE': 68,
      'ENOMSG': 49,
      'EIDRM': 24,
      'ECHRNG': 106,
      'EL2NSYNC': 156,
      'EL3HLT': 107,
      'EL3RST': 108,
      'ELNRNG': 109,
      'EUNATCH': 110,
      'ENOCSI': 111,
      'EL2HLT': 112,
      'EDEADLK': 16,
      'ENOLCK': 46,
      'EBADE': 113,
      'EBADR': 114,
      'EXFULL': 115,
      'ENOANO': 104,
      'EBADRQC': 103,
      'EBADSLT': 102,
      'EDEADLOCK': 16,
      'EBFONT': 101,
      'ENOSTR': 100,
      'ENODATA': 116,
      'ETIME': 117,
      'ENOSR': 118,
      'ENONET': 119,
      'ENOPKG': 120,
      'EREMOTE': 121,
      'ENOLINK': 47,
      'EADV': 122,
      'ESRMNT': 123,
      'ECOMM': 124,
      'EPROTO': 65,
      'EMULTIHOP': 36,
      'EDOTDOT': 125,
      'EBADMSG': 9,
      'ENOTUNIQ': 126,
      'EBADFD': 127,
      'EREMCHG': 128,
      'ELIBACC': 129,
      'ELIBBAD': 130,
      'ELIBSCN': 131,
      'ELIBMAX': 132,
      'ELIBEXEC': 133,
      'ENOSYS': 52,
      'ENOTEMPTY': 55,
      'ENAMETOOLONG': 37,
      'ELOOP': 32,
      'EOPNOTSUPP': 138,
      'EPFNOSUPPORT': 139,
      'ECONNRESET': 15,
      'ENOBUFS': 42,
      'EAFNOSUPPORT': 5,
      'EPROTOTYPE': 67,
      'ENOTSOCK': 57,
      'ENOPROTOOPT': 50,
      'ESHUTDOWN': 140,
      'ECONNREFUSED': 14,
      'EADDRINUSE': 3,
      'ECONNABORTED': 13,
      'ENETUNREACH': 40,
      'ENETDOWN': 38,
      'ETIMEDOUT': 73,
      'EHOSTDOWN': 142,
      'EHOSTUNREACH': 23,
      'EINPROGRESS': 26,
      'EALREADY': 7,
      'EDESTADDRREQ': 17,
      'EMSGSIZE': 35,
      'EPROTONOSUPPORT': 66,
      'ESOCKTNOSUPPORT': 137,
      'EADDRNOTAVAIL': 4,
      'ENETRESET': 39,
      'EISCONN': 30,
      'ENOTCONN': 53,
      'ETOOMANYREFS': 141,
      'EUSERS': 136,
      'EDQUOT': 19,
      'ESTALE': 72,
      'ENOTSUP': 138,
      'ENOMEDIUM': 148,
      'EILSEQ': 25,
      'EOVERFLOW': 61,
      'ECANCELED': 11,
      'ENOTRECOVERABLE': 56,
      'EOWNERDEAD': 62,
      'ESTRPIPE': 135,
    };
  var FS = {
  root:null,
  mounts:[],
  devices:{
  },
  streams:[],
  nextInode:1,
  nameTable:null,
  currentPath:"/",
  initialized:false,
  ignorePermissions:true,
  ErrnoError:class extends Error {
        // We set the `name` property to be able to identify `FS.ErrnoError`
        // - the `name` is a standard ECMA-262 property of error objects. Kind of good to have it anyway.
        // - when using PROXYFS, an error can come from an underlying FS
        // as different FS objects have their own FS.ErrnoError each,
        // the test `err instanceof FS.ErrnoError` won't detect an error coming from another filesystem, causing bugs.
        // we'll use the reliable test `err.name == "ErrnoError"` instead
        constructor(errno) {
          super(runtimeInitialized ? strError(errno) : '');
          // TODO(sbc): Use the inline member declaration syntax once we
          // support it in acorn and closure.
          this.name = 'ErrnoError';
          this.errno = errno;
          for (var key in ERRNO_CODES) {
            if (ERRNO_CODES[key] === errno) {
              this.code = key;
              break;
            }
          }
        }
      },
  genericErrors:{
  },
  filesystems:null,
  syncFSRequests:0,
  readFiles:{
  },
  FSStream:class {
        constructor() {
          // TODO(https://github.com/emscripten-core/emscripten/issues/21414):
          // Use inline field declarations.
          this.shared = {};
        }
        get object() {
          return this.node;
        }
        set object(val) {
          this.node = val;
        }
        get isRead() {
          return (this.flags & 2097155) !== 1;
        }
        get isWrite() {
          return (this.flags & 2097155) !== 0;
        }
        get isAppend() {
          return (this.flags & 1024);
        }
        get flags() {
          return this.shared.flags;
        }
        set flags(val) {
          this.shared.flags = val;
        }
        get position() {
          return this.shared.position;
        }
        set position(val) {
          this.shared.position = val;
        }
      },
  FSNode:class {
        constructor(parent, name, mode, rdev) {
          if (!parent) {
            parent = this;  // root node sets parent to itself
          }
          this.parent = parent;
          this.mount = parent.mount;
          this.mounted = null;
          this.id = FS.nextInode++;
          this.name = name;
          this.mode = mode;
          this.node_ops = {};
          this.stream_ops = {};
          this.rdev = rdev;
          this.readMode = 292 | 73;
          this.writeMode = 146;
        }
        get read() {
          return (this.mode & this.readMode) === this.readMode;
        }
        set read(val) {
          val ? this.mode |= this.readMode : this.mode &= ~this.readMode;
        }
        get write() {
          return (this.mode & this.writeMode) === this.writeMode;
        }
        set write(val) {
          val ? this.mode |= this.writeMode : this.mode &= ~this.writeMode;
        }
        get isFolder() {
          return FS.isDir(this.mode);
        }
        get isDevice() {
          return FS.isChrdev(this.mode);
        }
      },
  lookupPath(path, opts = {}) {
        path = PATH_FS.resolve(path);
  
        if (!path) return { path: '', node: null };
  
        var defaults = {
          follow_mount: true,
          recurse_count: 0
        };
        opts = Object.assign(defaults, opts)
  
        if (opts.recurse_count > 8) {  // max recursive lookup of 8
          throw new FS.ErrnoError(32);
        }
  
        // split the absolute path
        var parts = path.split('/').filter((p) => !!p);
  
        // start at the root
        var current = FS.root;
        var current_path = '/';
  
        for (var i = 0; i < parts.length; i++) {
          var islast = (i === parts.length-1);
          if (islast && opts.parent) {
            // stop resolving
            break;
          }
  
          current = FS.lookupNode(current, parts[i]);
          current_path = PATH.join2(current_path, parts[i]);
  
          // jump to the mount's root node if this is a mountpoint
          if (FS.isMountpoint(current)) {
            if (!islast || (islast && opts.follow_mount)) {
              current = current.mounted.root;
            }
          }
  
          // by default, lookupPath will not follow a symlink if it is the final path component.
          // setting opts.follow = true will override this behavior.
          if (!islast || opts.follow) {
            var count = 0;
            while (FS.isLink(current.mode)) {
              var link = FS.readlink(current_path);
              current_path = PATH_FS.resolve(PATH.dirname(current_path), link);
  
              var lookup = FS.lookupPath(current_path, { recurse_count: opts.recurse_count + 1 });
              current = lookup.node;
  
              if (count++ > 40) {  // limit max consecutive symlinks to 40 (SYMLOOP_MAX).
                throw new FS.ErrnoError(32);
              }
            }
          }
        }
  
        return { path: current_path, node: current };
      },
  getPath(node) {
        var path;
        while (true) {
          if (FS.isRoot(node)) {
            var mount = node.mount.mountpoint;
            if (!path) return mount;
            return mount[mount.length-1] !== '/' ? `${mount}/${path}` : mount + path;
          }
          path = path ? `${node.name}/${path}` : node.name;
          node = node.parent;
        }
      },
  hashName(parentid, name) {
        var hash = 0;
  
        for (var i = 0; i < name.length; i++) {
          hash = ((hash << 5) - hash + name.charCodeAt(i)) | 0;
        }
        return ((parentid + hash) >>> 0) % FS.nameTable.length;
      },
  hashAddNode(node) {
        var hash = FS.hashName(node.parent.id, node.name);
        node.name_next = FS.nameTable[hash];
        FS.nameTable[hash] = node;
      },
  hashRemoveNode(node) {
        var hash = FS.hashName(node.parent.id, node.name);
        if (FS.nameTable[hash] === node) {
          FS.nameTable[hash] = node.name_next;
        } else {
          var current = FS.nameTable[hash];
          while (current) {
            if (current.name_next === node) {
              current.name_next = node.name_next;
              break;
            }
            current = current.name_next;
          }
        }
      },
  lookupNode(parent, name) {
        var errCode = FS.mayLookup(parent);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        var hash = FS.hashName(parent.id, name);
        for (var node = FS.nameTable[hash]; node; node = node.name_next) {
          var nodeName = node.name;
          if (node.parent.id === parent.id && nodeName === name) {
            return node;
          }
        }
        // if we failed to find it in the cache, call into the VFS
        return FS.lookup(parent, name);
      },
  createNode(parent, name, mode, rdev) {
        assert(typeof parent == 'object')
        var node = new FS.FSNode(parent, name, mode, rdev);
  
        FS.hashAddNode(node);
  
        return node;
      },
  destroyNode(node) {
        FS.hashRemoveNode(node);
      },
  isRoot(node) {
        return node === node.parent;
      },
  isMountpoint(node) {
        return !!node.mounted;
      },
  isFile(mode) {
        return (mode & 61440) === 32768;
      },
  isDir(mode) {
        return (mode & 61440) === 16384;
      },
  isLink(mode) {
        return (mode & 61440) === 40960;
      },
  isChrdev(mode) {
        return (mode & 61440) === 8192;
      },
  isBlkdev(mode) {
        return (mode & 61440) === 24576;
      },
  isFIFO(mode) {
        return (mode & 61440) === 4096;
      },
  isSocket(mode) {
        return (mode & 49152) === 49152;
      },
  flagsToPermissionString(flag) {
        var perms = ['r', 'w', 'rw'][flag & 3];
        if ((flag & 512)) {
          perms += 'w';
        }
        return perms;
      },
  nodePermissions(node, perms) {
        if (FS.ignorePermissions) {
          return 0;
        }
        // return 0 if any user, group or owner bits are set.
        if (perms.includes('r') && !(node.mode & 292)) {
          return 2;
        } else if (perms.includes('w') && !(node.mode & 146)) {
          return 2;
        } else if (perms.includes('x') && !(node.mode & 73)) {
          return 2;
        }
        return 0;
      },
  mayLookup(dir) {
        if (!FS.isDir(dir.mode)) return 54;
        var errCode = FS.nodePermissions(dir, 'x');
        if (errCode) return errCode;
        if (!dir.node_ops.lookup) return 2;
        return 0;
      },
  mayCreate(dir, name) {
        try {
          var node = FS.lookupNode(dir, name);
          return 20;
        } catch (e) {
        }
        return FS.nodePermissions(dir, 'wx');
      },
  mayDelete(dir, name, isdir) {
        var node;
        try {
          node = FS.lookupNode(dir, name);
        } catch (e) {
          return e.errno;
        }
        var errCode = FS.nodePermissions(dir, 'wx');
        if (errCode) {
          return errCode;
        }
        if (isdir) {
          if (!FS.isDir(node.mode)) {
            return 54;
          }
          if (FS.isRoot(node) || FS.getPath(node) === FS.cwd()) {
            return 10;
          }
        } else {
          if (FS.isDir(node.mode)) {
            return 31;
          }
        }
        return 0;
      },
  mayOpen(node, flags) {
        if (!node) {
          return 44;
        }
        if (FS.isLink(node.mode)) {
          return 32;
        } else if (FS.isDir(node.mode)) {
          if (FS.flagsToPermissionString(flags) !== 'r' || // opening for write
              (flags & 512)) { // TODO: check for O_SEARCH? (== search for dir only)
            return 31;
          }
        }
        return FS.nodePermissions(node, FS.flagsToPermissionString(flags));
      },
  MAX_OPEN_FDS:4096,
  nextfd() {
        for (var fd = 0; fd <= FS.MAX_OPEN_FDS; fd++) {
          if (!FS.streams[fd]) {
            return fd;
          }
        }
        throw new FS.ErrnoError(33);
      },
  getStreamChecked(fd) {
        var stream = FS.getStream(fd);
        if (!stream) {
          throw new FS.ErrnoError(8);
        }
        return stream;
      },
  getStream:(fd) => FS.streams[fd],
  createStream(stream, fd = -1) {
        assert(fd >= -1);
  
        // clone it, so we can return an instance of FSStream
        stream = Object.assign(new FS.FSStream(), stream);
        if (fd == -1) {
          fd = FS.nextfd();
        }
        stream.fd = fd;
        FS.streams[fd] = stream;
        return stream;
      },
  closeStream(fd) {
        FS.streams[fd] = null;
      },
  dupStream(origStream, fd = -1) {
        var stream = FS.createStream(origStream, fd);
        stream.stream_ops?.dup?.(stream);
        return stream;
      },
  chrdev_stream_ops:{
  open(stream) {
          var device = FS.getDevice(stream.node.rdev);
          // override node's stream ops with the device's
          stream.stream_ops = device.stream_ops;
          // forward the open call
          stream.stream_ops.open?.(stream);
        },
  llseek() {
          throw new FS.ErrnoError(70);
        },
  },
  major:(dev) => ((dev) >> 8),
  minor:(dev) => ((dev) & 0xff),
  makedev:(ma, mi) => ((ma) << 8 | (mi)),
  registerDevice(dev, ops) {
        FS.devices[dev] = { stream_ops: ops };
      },
  getDevice:(dev) => FS.devices[dev],
  getMounts(mount) {
        var mounts = [];
        var check = [mount];
  
        while (check.length) {
          var m = check.pop();
  
          mounts.push(m);
  
          check.push(...m.mounts);
        }
  
        return mounts;
      },
  syncfs(populate, callback) {
        if (typeof populate == 'function') {
          callback = populate;
          populate = false;
        }
  
        FS.syncFSRequests++;
  
        if (FS.syncFSRequests > 1) {
          err(`warning: ${FS.syncFSRequests} FS.syncfs operations in flight at once, probably just doing extra work`);
        }
  
        var mounts = FS.getMounts(FS.root.mount);
        var completed = 0;
  
        function doCallback(errCode) {
          assert(FS.syncFSRequests > 0);
          FS.syncFSRequests--;
          return callback(errCode);
        }
  
        function done(errCode) {
          if (errCode) {
            if (!done.errored) {
              done.errored = true;
              return doCallback(errCode);
            }
            return;
          }
          if (++completed >= mounts.length) {
            doCallback(null);
          }
        };
  
        // sync all mounts
        mounts.forEach((mount) => {
          if (!mount.type.syncfs) {
            return done(null);
          }
          mount.type.syncfs(mount, populate, done);
        });
      },
  mount(type, opts, mountpoint) {
        if (typeof type == 'string') {
          // The filesystem was not included, and instead we have an error
          // message stored in the variable.
          throw type;
        }
        var root = mountpoint === '/';
        var pseudo = !mountpoint;
        var node;
  
        if (root && FS.root) {
          throw new FS.ErrnoError(10);
        } else if (!root && !pseudo) {
          var lookup = FS.lookupPath(mountpoint, { follow_mount: false });
  
          mountpoint = lookup.path;  // use the absolute path
          node = lookup.node;
  
          if (FS.isMountpoint(node)) {
            throw new FS.ErrnoError(10);
          }
  
          if (!FS.isDir(node.mode)) {
            throw new FS.ErrnoError(54);
          }
        }
  
        var mount = {
          type,
          opts,
          mountpoint,
          mounts: []
        };
  
        // create a root node for the fs
        var mountRoot = type.mount(mount);
        mountRoot.mount = mount;
        mount.root = mountRoot;
  
        if (root) {
          FS.root = mountRoot;
        } else if (node) {
          // set as a mountpoint
          node.mounted = mount;
  
          // add the new mount to the current mount's children
          if (node.mount) {
            node.mount.mounts.push(mount);
          }
        }
  
        return mountRoot;
      },
  unmount(mountpoint) {
        var lookup = FS.lookupPath(mountpoint, { follow_mount: false });
  
        if (!FS.isMountpoint(lookup.node)) {
          throw new FS.ErrnoError(28);
        }
  
        // destroy the nodes for this mount, and all its child mounts
        var node = lookup.node;
        var mount = node.mounted;
        var mounts = FS.getMounts(mount);
  
        Object.keys(FS.nameTable).forEach((hash) => {
          var current = FS.nameTable[hash];
  
          while (current) {
            var next = current.name_next;
  
            if (mounts.includes(current.mount)) {
              FS.destroyNode(current);
            }
  
            current = next;
          }
        });
  
        // no longer a mountpoint
        node.mounted = null;
  
        // remove this mount from the child mounts
        var idx = node.mount.mounts.indexOf(mount);
        assert(idx !== -1);
        node.mount.mounts.splice(idx, 1);
      },
  lookup(parent, name) {
        return parent.node_ops.lookup(parent, name);
      },
  mknod(path, mode, dev) {
        var lookup = FS.lookupPath(path, { parent: true });
        var parent = lookup.node;
        var name = PATH.basename(path);
        if (!name || name === '.' || name === '..') {
          throw new FS.ErrnoError(28);
        }
        var errCode = FS.mayCreate(parent, name);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        if (!parent.node_ops.mknod) {
          throw new FS.ErrnoError(63);
        }
        return parent.node_ops.mknod(parent, name, mode, dev);
      },
  create(path, mode) {
        mode = mode !== undefined ? mode : 438 /* 0666 */;
        mode &= 4095;
        mode |= 32768;
        return FS.mknod(path, mode, 0);
      },
  mkdir(path, mode) {
        mode = mode !== undefined ? mode : 511 /* 0777 */;
        mode &= 511 | 512;
        mode |= 16384;
        return FS.mknod(path, mode, 0);
      },
  mkdirTree(path, mode) {
        var dirs = path.split('/');
        var d = '';
        for (var i = 0; i < dirs.length; ++i) {
          if (!dirs[i]) continue;
          d += '/' + dirs[i];
          try {
            FS.mkdir(d, mode);
          } catch(e) {
            if (e.errno != 20) throw e;
          }
        }
      },
  mkdev(path, mode, dev) {
        if (typeof dev == 'undefined') {
          dev = mode;
          mode = 438 /* 0666 */;
        }
        mode |= 8192;
        return FS.mknod(path, mode, dev);
      },
  symlink(oldpath, newpath) {
        if (!PATH_FS.resolve(oldpath)) {
          throw new FS.ErrnoError(44);
        }
        var lookup = FS.lookupPath(newpath, { parent: true });
        var parent = lookup.node;
        if (!parent) {
          throw new FS.ErrnoError(44);
        }
        var newname = PATH.basename(newpath);
        var errCode = FS.mayCreate(parent, newname);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        if (!parent.node_ops.symlink) {
          throw new FS.ErrnoError(63);
        }
        return parent.node_ops.symlink(parent, newname, oldpath);
      },
  rename(old_path, new_path) {
        var old_dirname = PATH.dirname(old_path);
        var new_dirname = PATH.dirname(new_path);
        var old_name = PATH.basename(old_path);
        var new_name = PATH.basename(new_path);
        // parents must exist
        var lookup, old_dir, new_dir;
  
        // let the errors from non existent directories percolate up
        lookup = FS.lookupPath(old_path, { parent: true });
        old_dir = lookup.node;
        lookup = FS.lookupPath(new_path, { parent: true });
        new_dir = lookup.node;
  
        if (!old_dir || !new_dir) throw new FS.ErrnoError(44);
        // need to be part of the same mount
        if (old_dir.mount !== new_dir.mount) {
          throw new FS.ErrnoError(75);
        }
        // source must exist
        var old_node = FS.lookupNode(old_dir, old_name);
        // old path should not be an ancestor of the new path
        var relative = PATH_FS.relative(old_path, new_dirname);
        if (relative.charAt(0) !== '.') {
          throw new FS.ErrnoError(28);
        }
        // new path should not be an ancestor of the old path
        relative = PATH_FS.relative(new_path, old_dirname);
        if (relative.charAt(0) !== '.') {
          throw new FS.ErrnoError(55);
        }
        // see if the new path already exists
        var new_node;
        try {
          new_node = FS.lookupNode(new_dir, new_name);
        } catch (e) {
          // not fatal
        }
        // early out if nothing needs to change
        if (old_node === new_node) {
          return;
        }
        // we'll need to delete the old entry
        var isdir = FS.isDir(old_node.mode);
        var errCode = FS.mayDelete(old_dir, old_name, isdir);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        // need delete permissions if we'll be overwriting.
        // need create permissions if new doesn't already exist.
        errCode = new_node ?
          FS.mayDelete(new_dir, new_name, isdir) :
          FS.mayCreate(new_dir, new_name);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        if (!old_dir.node_ops.rename) {
          throw new FS.ErrnoError(63);
        }
        if (FS.isMountpoint(old_node) || (new_node && FS.isMountpoint(new_node))) {
          throw new FS.ErrnoError(10);
        }
        // if we are going to change the parent, check write permissions
        if (new_dir !== old_dir) {
          errCode = FS.nodePermissions(old_dir, 'w');
          if (errCode) {
            throw new FS.ErrnoError(errCode);
          }
        }
        // remove the node from the lookup hash
        FS.hashRemoveNode(old_node);
        // do the underlying fs rename
        try {
          old_dir.node_ops.rename(old_node, new_dir, new_name);
          // update old node (we do this here to avoid each backend 
          // needing to)
          old_node.parent = new_dir;
        } catch (e) {
          throw e;
        } finally {
          // add the node back to the hash (in case node_ops.rename
          // changed its name)
          FS.hashAddNode(old_node);
        }
      },
  rmdir(path) {
        var lookup = FS.lookupPath(path, { parent: true });
        var parent = lookup.node;
        var name = PATH.basename(path);
        var node = FS.lookupNode(parent, name);
        var errCode = FS.mayDelete(parent, name, true);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        if (!parent.node_ops.rmdir) {
          throw new FS.ErrnoError(63);
        }
        if (FS.isMountpoint(node)) {
          throw new FS.ErrnoError(10);
        }
        parent.node_ops.rmdir(parent, name);
        FS.destroyNode(node);
      },
  readdir(path) {
        var lookup = FS.lookupPath(path, { follow: true });
        var node = lookup.node;
        if (!node.node_ops.readdir) {
          throw new FS.ErrnoError(54);
        }
        return node.node_ops.readdir(node);
      },
  unlink(path) {
        var lookup = FS.lookupPath(path, { parent: true });
        var parent = lookup.node;
        if (!parent) {
          throw new FS.ErrnoError(44);
        }
        var name = PATH.basename(path);
        var node = FS.lookupNode(parent, name);
        var errCode = FS.mayDelete(parent, name, false);
        if (errCode) {
          // According to POSIX, we should map EISDIR to EPERM, but
          // we instead do what Linux does (and we must, as we use
          // the musl linux libc).
          throw new FS.ErrnoError(errCode);
        }
        if (!parent.node_ops.unlink) {
          throw new FS.ErrnoError(63);
        }
        if (FS.isMountpoint(node)) {
          throw new FS.ErrnoError(10);
        }
        parent.node_ops.unlink(parent, name);
        FS.destroyNode(node);
      },
  readlink(path) {
        var lookup = FS.lookupPath(path);
        var link = lookup.node;
        if (!link) {
          throw new FS.ErrnoError(44);
        }
        if (!link.node_ops.readlink) {
          throw new FS.ErrnoError(28);
        }
        return PATH_FS.resolve(FS.getPath(link.parent), link.node_ops.readlink(link));
      },
  stat(path, dontFollow) {
        var lookup = FS.lookupPath(path, { follow: !dontFollow });
        var node = lookup.node;
        if (!node) {
          throw new FS.ErrnoError(44);
        }
        if (!node.node_ops.getattr) {
          throw new FS.ErrnoError(63);
        }
        return node.node_ops.getattr(node);
      },
  lstat(path) {
        return FS.stat(path, true);
      },
  chmod(path, mode, dontFollow) {
        var node;
        if (typeof path == 'string') {
          var lookup = FS.lookupPath(path, { follow: !dontFollow });
          node = lookup.node;
        } else {
          node = path;
        }
        if (!node.node_ops.setattr) {
          throw new FS.ErrnoError(63);
        }
        node.node_ops.setattr(node, {
          mode: (mode & 4095) | (node.mode & ~4095),
          timestamp: Date.now()
        });
      },
  lchmod(path, mode) {
        FS.chmod(path, mode, true);
      },
  fchmod(fd, mode) {
        var stream = FS.getStreamChecked(fd);
        FS.chmod(stream.node, mode);
      },
  chown(path, uid, gid, dontFollow) {
        var node;
        if (typeof path == 'string') {
          var lookup = FS.lookupPath(path, { follow: !dontFollow });
          node = lookup.node;
        } else {
          node = path;
        }
        if (!node.node_ops.setattr) {
          throw new FS.ErrnoError(63);
        }
        node.node_ops.setattr(node, {
          timestamp: Date.now()
          // we ignore the uid / gid for now
        });
      },
  lchown(path, uid, gid) {
        FS.chown(path, uid, gid, true);
      },
  fchown(fd, uid, gid) {
        var stream = FS.getStreamChecked(fd);
        FS.chown(stream.node, uid, gid);
      },
  truncate(path, len) {
        if (len < 0) {
          throw new FS.ErrnoError(28);
        }
        var node;
        if (typeof path == 'string') {
          var lookup = FS.lookupPath(path, { follow: true });
          node = lookup.node;
        } else {
          node = path;
        }
        if (!node.node_ops.setattr) {
          throw new FS.ErrnoError(63);
        }
        if (FS.isDir(node.mode)) {
          throw new FS.ErrnoError(31);
        }
        if (!FS.isFile(node.mode)) {
          throw new FS.ErrnoError(28);
        }
        var errCode = FS.nodePermissions(node, 'w');
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        node.node_ops.setattr(node, {
          size: len,
          timestamp: Date.now()
        });
      },
  ftruncate(fd, len) {
        var stream = FS.getStreamChecked(fd);
        if ((stream.flags & 2097155) === 0) {
          throw new FS.ErrnoError(28);
        }
        FS.truncate(stream.node, len);
      },
  utime(path, atime, mtime) {
        var lookup = FS.lookupPath(path, { follow: true });
        var node = lookup.node;
        node.node_ops.setattr(node, {
          timestamp: Math.max(atime, mtime)
        });
      },
  open(path, flags, mode) {
        if (path === "") {
          throw new FS.ErrnoError(44);
        }
        flags = typeof flags == 'string' ? FS_modeStringToFlags(flags) : flags;
        if ((flags & 64)) {
          mode = typeof mode == 'undefined' ? 438 /* 0666 */ : mode;
          mode = (mode & 4095) | 32768;
        } else {
          mode = 0;
        }
        var node;
        if (typeof path == 'object') {
          node = path;
        } else {
          path = PATH.normalize(path);
          try {
            var lookup = FS.lookupPath(path, {
              follow: !(flags & 131072)
            });
            node = lookup.node;
          } catch (e) {
            // ignore
          }
        }
        // perhaps we need to create the node
        var created = false;
        if ((flags & 64)) {
          if (node) {
            // if O_CREAT and O_EXCL are set, error out if the node already exists
            if ((flags & 128)) {
              throw new FS.ErrnoError(20);
            }
          } else {
            // node doesn't exist, try to create it
            node = FS.mknod(path, mode, 0);
            created = true;
          }
        }
        if (!node) {
          throw new FS.ErrnoError(44);
        }
        // can't truncate a device
        if (FS.isChrdev(node.mode)) {
          flags &= ~512;
        }
        // if asked only for a directory, then this must be one
        if ((flags & 65536) && !FS.isDir(node.mode)) {
          throw new FS.ErrnoError(54);
        }
        // check permissions, if this is not a file we just created now (it is ok to
        // create and write to a file with read-only permissions; it is read-only
        // for later use)
        if (!created) {
          var errCode = FS.mayOpen(node, flags);
          if (errCode) {
            throw new FS.ErrnoError(errCode);
          }
        }
        // do truncation if necessary
        if ((flags & 512) && !created) {
          FS.truncate(node, 0);
        }
        // we've already handled these, don't pass down to the underlying vfs
        flags &= ~(128 | 512 | 131072);
  
        // register the stream with the filesystem
        var stream = FS.createStream({
          node,
          path: FS.getPath(node),  // we want the absolute path to the node
          flags,
          seekable: true,
          position: 0,
          stream_ops: node.stream_ops,
          // used by the file family libc calls (fopen, fwrite, ferror, etc.)
          ungotten: [],
          error: false
        });
        // call the new stream's open function
        if (stream.stream_ops.open) {
          stream.stream_ops.open(stream);
        }
        if (Module['logReadFiles'] && !(flags & 1)) {
          if (!(path in FS.readFiles)) {
            FS.readFiles[path] = 1;
          }
        }
        return stream;
      },
  close(stream) {
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if (stream.getdents) stream.getdents = null; // free readdir state
        try {
          if (stream.stream_ops.close) {
            stream.stream_ops.close(stream);
          }
        } catch (e) {
          throw e;
        } finally {
          FS.closeStream(stream.fd);
        }
        stream.fd = null;
      },
  isClosed(stream) {
        return stream.fd === null;
      },
  llseek(stream, offset, whence) {
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if (!stream.seekable || !stream.stream_ops.llseek) {
          throw new FS.ErrnoError(70);
        }
        if (whence != 0 && whence != 1 && whence != 2) {
          throw new FS.ErrnoError(28);
        }
        stream.position = stream.stream_ops.llseek(stream, offset, whence);
        stream.ungotten = [];
        return stream.position;
      },
  read(stream, buffer, offset, length, position) {
        assert(offset >= 0);
        if (length < 0 || position < 0) {
          throw new FS.ErrnoError(28);
        }
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if ((stream.flags & 2097155) === 1) {
          throw new FS.ErrnoError(8);
        }
        if (FS.isDir(stream.node.mode)) {
          throw new FS.ErrnoError(31);
        }
        if (!stream.stream_ops.read) {
          throw new FS.ErrnoError(28);
        }
        var seeking = typeof position != 'undefined';
        if (!seeking) {
          position = stream.position;
        } else if (!stream.seekable) {
          throw new FS.ErrnoError(70);
        }
        var bytesRead = stream.stream_ops.read(stream, buffer, offset, length, position);
        if (!seeking) stream.position += bytesRead;
        return bytesRead;
      },
  write(stream, buffer, offset, length, position, canOwn) {
        assert(offset >= 0);
        if (length < 0 || position < 0) {
          throw new FS.ErrnoError(28);
        }
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if ((stream.flags & 2097155) === 0) {
          throw new FS.ErrnoError(8);
        }
        if (FS.isDir(stream.node.mode)) {
          throw new FS.ErrnoError(31);
        }
        if (!stream.stream_ops.write) {
          throw new FS.ErrnoError(28);
        }
        if (stream.seekable && stream.flags & 1024) {
          // seek to the end before writing in append mode
          FS.llseek(stream, 0, 2);
        }
        var seeking = typeof position != 'undefined';
        if (!seeking) {
          position = stream.position;
        } else if (!stream.seekable) {
          throw new FS.ErrnoError(70);
        }
        var bytesWritten = stream.stream_ops.write(stream, buffer, offset, length, position, canOwn);
        if (!seeking) stream.position += bytesWritten;
        return bytesWritten;
      },
  allocate(stream, offset, length) {
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if (offset < 0 || length <= 0) {
          throw new FS.ErrnoError(28);
        }
        if ((stream.flags & 2097155) === 0) {
          throw new FS.ErrnoError(8);
        }
        if (!FS.isFile(stream.node.mode) && !FS.isDir(stream.node.mode)) {
          throw new FS.ErrnoError(43);
        }
        if (!stream.stream_ops.allocate) {
          throw new FS.ErrnoError(138);
        }
        stream.stream_ops.allocate(stream, offset, length);
      },
  mmap(stream, length, position, prot, flags) {
        // User requests writing to file (prot & PROT_WRITE != 0).
        // Checking if we have permissions to write to the file unless
        // MAP_PRIVATE flag is set. According to POSIX spec it is possible
        // to write to file opened in read-only mode with MAP_PRIVATE flag,
        // as all modifications will be visible only in the memory of
        // the current process.
        if ((prot & 2) !== 0
            && (flags & 2) === 0
            && (stream.flags & 2097155) !== 2) {
          throw new FS.ErrnoError(2);
        }
        if ((stream.flags & 2097155) === 1) {
          throw new FS.ErrnoError(2);
        }
        if (!stream.stream_ops.mmap) {
          throw new FS.ErrnoError(43);
        }
        if (!length) {
          throw new FS.ErrnoError(28);
        }
        return stream.stream_ops.mmap(stream, length, position, prot, flags);
      },
  msync(stream, buffer, offset, length, mmapFlags) {
        assert(offset >= 0);
        if (!stream.stream_ops.msync) {
          return 0;
        }
        return stream.stream_ops.msync(stream, buffer, offset, length, mmapFlags);
      },
  ioctl(stream, cmd, arg) {
        if (!stream.stream_ops.ioctl) {
          throw new FS.ErrnoError(59);
        }
        return stream.stream_ops.ioctl(stream, cmd, arg);
      },
  readFile(path, opts = {}) {
        opts.flags = opts.flags || 0;
        opts.encoding = opts.encoding || 'binary';
        if (opts.encoding !== 'utf8' && opts.encoding !== 'binary') {
          throw new Error(`Invalid encoding type "${opts.encoding}"`);
        }
        var ret;
        var stream = FS.open(path, opts.flags);
        var stat = FS.stat(path);
        var length = stat.size;
        var buf = new Uint8Array(length);
        FS.read(stream, buf, 0, length, 0);
        if (opts.encoding === 'utf8') {
          ret = UTF8ArrayToString(buf);
        } else if (opts.encoding === 'binary') {
          ret = buf;
        }
        FS.close(stream);
        return ret;
      },
  writeFile(path, data, opts = {}) {
        opts.flags = opts.flags || 577;
        var stream = FS.open(path, opts.flags, opts.mode);
        if (typeof data == 'string') {
          var buf = new Uint8Array(lengthBytesUTF8(data)+1);
          var actualNumBytes = stringToUTF8Array(data, buf, 0, buf.length);
          FS.write(stream, buf, 0, actualNumBytes, undefined, opts.canOwn);
        } else if (ArrayBuffer.isView(data)) {
          FS.write(stream, data, 0, data.byteLength, undefined, opts.canOwn);
        } else {
          throw new Error('Unsupported data type');
        }
        FS.close(stream);
      },
  cwd:() => FS.currentPath,
  chdir(path) {
        var lookup = FS.lookupPath(path, { follow: true });
        if (lookup.node === null) {
          throw new FS.ErrnoError(44);
        }
        if (!FS.isDir(lookup.node.mode)) {
          throw new FS.ErrnoError(54);
        }
        var errCode = FS.nodePermissions(lookup.node, 'x');
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        FS.currentPath = lookup.path;
      },
  createDefaultDirectories() {
        FS.mkdir('/tmp');
        FS.mkdir('/home');
        FS.mkdir('/home/web_user');
      },
  createDefaultDevices() {
        // create /dev
        FS.mkdir('/dev');
        // setup /dev/null
        FS.registerDevice(FS.makedev(1, 3), {
          read: () => 0,
          write: (stream, buffer, offset, length, pos) => length,
        });
        FS.mkdev('/dev/null', FS.makedev(1, 3));
        // setup /dev/tty and /dev/tty1
        // stderr needs to print output using err() rather than out()
        // so we register a second tty just for it.
        TTY.register(FS.makedev(5, 0), TTY.default_tty_ops);
        TTY.register(FS.makedev(6, 0), TTY.default_tty1_ops);
        FS.mkdev('/dev/tty', FS.makedev(5, 0));
        FS.mkdev('/dev/tty1', FS.makedev(6, 0));
        // setup /dev/[u]random
        // use a buffer to avoid overhead of individual crypto calls per byte
        var randomBuffer = new Uint8Array(1024), randomLeft = 0;
        var randomByte = () => {
          if (randomLeft === 0) {
            randomLeft = randomFill(randomBuffer).byteLength;
          }
          return randomBuffer[--randomLeft];
        };
        FS.createDevice('/dev', 'random', randomByte);
        FS.createDevice('/dev', 'urandom', randomByte);
        // we're not going to emulate the actual shm device,
        // just create the tmp dirs that reside in it commonly
        FS.mkdir('/dev/shm');
        FS.mkdir('/dev/shm/tmp');
      },
  createSpecialDirectories() {
        // create /proc/self/fd which allows /proc/self/fd/6 => readlink gives the
        // name of the stream for fd 6 (see test_unistd_ttyname)
        FS.mkdir('/proc');
        var proc_self = FS.mkdir('/proc/self');
        FS.mkdir('/proc/self/fd');
        FS.mount({
          mount() {
            var node = FS.createNode(proc_self, 'fd', 16384 | 511 /* 0777 */, 73);
            node.node_ops = {
              lookup(parent, name) {
                var fd = +name;
                var stream = FS.getStreamChecked(fd);
                var ret = {
                  parent: null,
                  mount: { mountpoint: 'fake' },
                  node_ops: { readlink: () => stream.path },
                };
                ret.parent = ret; // make it look like a simple root node
                return ret;
              }
            };
            return node;
          }
        }, {}, '/proc/self/fd');
      },
  createStandardStreams(input, output, error) {
        // TODO deprecate the old functionality of a single
        // input / output callback and that utilizes FS.createDevice
        // and instead require a unique set of stream ops
  
        // by default, we symlink the standard streams to the
        // default tty devices. however, if the standard streams
        // have been overwritten we create a unique device for
        // them instead.
        if (input) {
          FS.createDevice('/dev', 'stdin', input);
        } else {
          FS.symlink('/dev/tty', '/dev/stdin');
        }
        if (output) {
          FS.createDevice('/dev', 'stdout', null, output);
        } else {
          FS.symlink('/dev/tty', '/dev/stdout');
        }
        if (error) {
          FS.createDevice('/dev', 'stderr', null, error);
        } else {
          FS.symlink('/dev/tty1', '/dev/stderr');
        }
  
        // open default streams for the stdin, stdout and stderr devices
        var stdin = FS.open('/dev/stdin', 0);
        var stdout = FS.open('/dev/stdout', 1);
        var stderr = FS.open('/dev/stderr', 1);
        assert(stdin.fd === 0, `invalid handle for stdin (${stdin.fd})`);
        assert(stdout.fd === 1, `invalid handle for stdout (${stdout.fd})`);
        assert(stderr.fd === 2, `invalid handle for stderr (${stderr.fd})`);
      },
  staticInit() {
        // Some errors may happen quite a bit, to avoid overhead we reuse them (and suffer a lack of stack info)
        [44].forEach((code) => {
          FS.genericErrors[code] = new FS.ErrnoError(code);
          FS.genericErrors[code].stack = '<generic error, no stack>';
        });
  
        FS.nameTable = new Array(4096);
  
        FS.mount(MEMFS, {}, '/');
  
        FS.createDefaultDirectories();
        FS.createDefaultDevices();
        FS.createSpecialDirectories();
  
        FS.filesystems = {
          'MEMFS': MEMFS,
        };
      },
  init(input, output, error) {
        assert(!FS.initialized, 'FS.init was previously called. If you want to initialize later with custom parameters, remove any earlier calls (note that one is automatically added to the generated code)');
        FS.initialized = true;
  
        // Allow Module.stdin etc. to provide defaults, if none explicitly passed to us here
        input ??= Module['stdin'];
        output ??= Module['stdout'];
        error ??= Module['stderr'];
  
        FS.createStandardStreams(input, output, error);
      },
  quit() {
        FS.initialized = false;
        // force-flush all streams, so we get musl std streams printed out
        _fflush(0);
        // close all of our streams
        for (var i = 0; i < FS.streams.length; i++) {
          var stream = FS.streams[i];
          if (!stream) {
            continue;
          }
          FS.close(stream);
        }
      },
  findObject(path, dontResolveLastLink) {
        var ret = FS.analyzePath(path, dontResolveLastLink);
        if (!ret.exists) {
          return null;
        }
        return ret.object;
      },
  analyzePath(path, dontResolveLastLink) {
        // operate from within the context of the symlink's target
        try {
          var lookup = FS.lookupPath(path, { follow: !dontResolveLastLink });
          path = lookup.path;
        } catch (e) {
        }
        var ret = {
          isRoot: false, exists: false, error: 0, name: null, path: null, object: null,
          parentExists: false, parentPath: null, parentObject: null
        };
        try {
          var lookup = FS.lookupPath(path, { parent: true });
          ret.parentExists = true;
          ret.parentPath = lookup.path;
          ret.parentObject = lookup.node;
          ret.name = PATH.basename(path);
          lookup = FS.lookupPath(path, { follow: !dontResolveLastLink });
          ret.exists = true;
          ret.path = lookup.path;
          ret.object = lookup.node;
          ret.name = lookup.node.name;
          ret.isRoot = lookup.path === '/';
        } catch (e) {
          ret.error = e.errno;
        };
        return ret;
      },
  createPath(parent, path, canRead, canWrite) {
        parent = typeof parent == 'string' ? parent : FS.getPath(parent);
        var parts = path.split('/').reverse();
        while (parts.length) {
          var part = parts.pop();
          if (!part) continue;
          var current = PATH.join2(parent, part);
          try {
            FS.mkdir(current);
          } catch (e) {
            // ignore EEXIST
          }
          parent = current;
        }
        return current;
      },
  createFile(parent, name, properties, canRead, canWrite) {
        var path = PATH.join2(typeof parent == 'string' ? parent : FS.getPath(parent), name);
        var mode = FS_getMode(canRead, canWrite);
        return FS.create(path, mode);
      },
  createDataFile(parent, name, data, canRead, canWrite, canOwn) {
        var path = name;
        if (parent) {
          parent = typeof parent == 'string' ? parent : FS.getPath(parent);
          path = name ? PATH.join2(parent, name) : parent;
        }
        var mode = FS_getMode(canRead, canWrite);
        var node = FS.create(path, mode);
        if (data) {
          if (typeof data == 'string') {
            var arr = new Array(data.length);
            for (var i = 0, len = data.length; i < len; ++i) arr[i] = data.charCodeAt(i);
            data = arr;
          }
          // make sure we can write to the file
          FS.chmod(node, mode | 146);
          var stream = FS.open(node, 577);
          FS.write(stream, data, 0, data.length, 0, canOwn);
          FS.close(stream);
          FS.chmod(node, mode);
        }
      },
  createDevice(parent, name, input, output) {
        var path = PATH.join2(typeof parent == 'string' ? parent : FS.getPath(parent), name);
        var mode = FS_getMode(!!input, !!output);
        FS.createDevice.major ??= 64;
        var dev = FS.makedev(FS.createDevice.major++, 0);
        // Create a fake device that a set of stream ops to emulate
        // the old behavior.
        FS.registerDevice(dev, {
          open(stream) {
            stream.seekable = false;
          },
          close(stream) {
            // flush any pending line data
            if (output?.buffer?.length) {
              output(10);
            }
          },
          read(stream, buffer, offset, length, pos /* ignored */) {
            var bytesRead = 0;
            for (var i = 0; i < length; i++) {
              var result;
              try {
                result = input();
              } catch (e) {
                throw new FS.ErrnoError(29);
              }
              if (result === undefined && bytesRead === 0) {
                throw new FS.ErrnoError(6);
              }
              if (result === null || result === undefined) break;
              bytesRead++;
              buffer[offset+i] = result;
            }
            if (bytesRead) {
              stream.node.timestamp = Date.now();
            }
            return bytesRead;
          },
          write(stream, buffer, offset, length, pos) {
            for (var i = 0; i < length; i++) {
              try {
                output(buffer[offset+i]);
              } catch (e) {
                throw new FS.ErrnoError(29);
              }
            }
            if (length) {
              stream.node.timestamp = Date.now();
            }
            return i;
          }
        });
        return FS.mkdev(path, mode, dev);
      },
  forceLoadFile(obj) {
        if (obj.isDevice || obj.isFolder || obj.link || obj.contents) return true;
        if (typeof XMLHttpRequest != 'undefined') {
          throw new Error("Lazy loading should have been performed (contents set) in createLazyFile, but it was not. Lazy loading only works in web workers. Use --embed-file or --preload-file in emcc on the main thread.");
        } else { // Command-line.
          try {
            obj.contents = readBinary(obj.url);
            obj.usedBytes = obj.contents.length;
          } catch (e) {
            throw new FS.ErrnoError(29);
          }
        }
      },
  createLazyFile(parent, name, url, canRead, canWrite) {
        // Lazy chunked Uint8Array (implements get and length from Uint8Array).
        // Actual getting is abstracted away for eventual reuse.
        class LazyUint8Array {
          constructor() {
            this.lengthKnown = false;
            this.chunks = []; // Loaded chunks. Index is the chunk number
          }
          get(idx) {
            if (idx > this.length-1 || idx < 0) {
              return undefined;
            }
            var chunkOffset = idx % this.chunkSize;
            var chunkNum = (idx / this.chunkSize)|0;
            return this.getter(chunkNum)[chunkOffset];
          }
          setDataGetter(getter) {
            this.getter = getter;
          }
          cacheLength() {
            // Find length
            var xhr = new XMLHttpRequest();
            xhr.open('HEAD', url, false);
            xhr.send(null);
            if (!(xhr.status >= 200 && xhr.status < 300 || xhr.status === 304)) throw new Error("Couldn't load " + url + ". Status: " + xhr.status);
            var datalength = Number(xhr.getResponseHeader("Content-length"));
            var header;
            var hasByteServing = (header = xhr.getResponseHeader("Accept-Ranges")) && header === "bytes";
            var usesGzip = (header = xhr.getResponseHeader("Content-Encoding")) && header === "gzip";
  
            var chunkSize = 1024*1024; // Chunk size in bytes
  
            if (!hasByteServing) chunkSize = datalength;
  
            // Function to get a range from the remote URL.
            var doXHR = (from, to) => {
              if (from > to) throw new Error("invalid range (" + from + ", " + to + ") or no bytes requested!");
              if (to > datalength-1) throw new Error("only " + datalength + " bytes available! programmer error!");
  
              // TODO: Use mozResponseArrayBuffer, responseStream, etc. if available.
              var xhr = new XMLHttpRequest();
              xhr.open('GET', url, false);
              if (datalength !== chunkSize) xhr.setRequestHeader("Range", "bytes=" + from + "-" + to);
  
              // Some hints to the browser that we want binary data.
              xhr.responseType = 'arraybuffer';
              if (xhr.overrideMimeType) {
                xhr.overrideMimeType('text/plain; charset=x-user-defined');
              }
  
              xhr.send(null);
              if (!(xhr.status >= 200 && xhr.status < 300 || xhr.status === 304)) throw new Error("Couldn't load " + url + ". Status: " + xhr.status);
              if (xhr.response !== undefined) {
                return new Uint8Array(/** @type{Array<number>} */(xhr.response || []));
              }
              return intArrayFromString(xhr.responseText || '', true);
            };
            var lazyArray = this;
            lazyArray.setDataGetter((chunkNum) => {
              var start = chunkNum * chunkSize;
              var end = (chunkNum+1) * chunkSize - 1; // including this byte
              end = Math.min(end, datalength-1); // if datalength-1 is selected, this is the last block
              if (typeof lazyArray.chunks[chunkNum] == 'undefined') {
                lazyArray.chunks[chunkNum] = doXHR(start, end);
              }
              if (typeof lazyArray.chunks[chunkNum] == 'undefined') throw new Error('doXHR failed!');
              return lazyArray.chunks[chunkNum];
            });
  
            if (usesGzip || !datalength) {
              // if the server uses gzip or doesn't supply the length, we have to download the whole file to get the (uncompressed) length
              chunkSize = datalength = 1; // this will force getter(0)/doXHR do download the whole file
              datalength = this.getter(0).length;
              chunkSize = datalength;
              out("LazyFiles on gzip forces download of the whole file when length is accessed");
            }
  
            this._length = datalength;
            this._chunkSize = chunkSize;
            this.lengthKnown = true;
          }
          get length() {
            if (!this.lengthKnown) {
              this.cacheLength();
            }
            return this._length;
          }
          get chunkSize() {
            if (!this.lengthKnown) {
              this.cacheLength();
            }
            return this._chunkSize;
          }
        }
  
        if (typeof XMLHttpRequest != 'undefined') {
          if (!ENVIRONMENT_IS_WORKER) throw 'Cannot do synchronous binary XHRs outside webworkers in modern browsers. Use --embed-file or --preload-file in emcc';
          var lazyArray = new LazyUint8Array();
          var properties = { isDevice: false, contents: lazyArray };
        } else {
          var properties = { isDevice: false, url: url };
        }
  
        var node = FS.createFile(parent, name, properties, canRead, canWrite);
        // This is a total hack, but I want to get this lazy file code out of the
        // core of MEMFS. If we want to keep this lazy file concept I feel it should
        // be its own thin LAZYFS proxying calls to MEMFS.
        if (properties.contents) {
          node.contents = properties.contents;
        } else if (properties.url) {
          node.contents = null;
          node.url = properties.url;
        }
        // Add a function that defers querying the file size until it is asked the first time.
        Object.defineProperties(node, {
          usedBytes: {
            get: function() { return this.contents.length; }
          }
        });
        // override each stream op with one that tries to force load the lazy file first
        var stream_ops = {};
        var keys = Object.keys(node.stream_ops);
        keys.forEach((key) => {
          var fn = node.stream_ops[key];
          stream_ops[key] = (...args) => {
            FS.forceLoadFile(node);
            return fn(...args);
          };
        });
        function writeChunks(stream, buffer, offset, length, position) {
          var contents = stream.node.contents;
          if (position >= contents.length)
            return 0;
          var size = Math.min(contents.length - position, length);
          assert(size >= 0);
          if (contents.slice) { // normal array
            for (var i = 0; i < size; i++) {
              buffer[offset + i] = contents[position + i];
            }
          } else {
            for (var i = 0; i < size; i++) { // LazyUint8Array from sync binary XHR
              buffer[offset + i] = contents.get(position + i);
            }
          }
          return size;
        }
        // use a custom read function
        stream_ops.read = (stream, buffer, offset, length, position) => {
          FS.forceLoadFile(node);
          return writeChunks(stream, buffer, offset, length, position)
        };
        // use a custom mmap function
        stream_ops.mmap = (stream, length, position, prot, flags) => {
          FS.forceLoadFile(node);
          var ptr = mmapAlloc(length);
          if (!ptr) {
            throw new FS.ErrnoError(48);
          }
          writeChunks(stream, HEAP8, ptr, length, position);
          return { ptr, allocated: true };
        };
        node.stream_ops = stream_ops;
        return node;
      },
  absolutePath() {
        abort('FS.absolutePath has been removed; use PATH_FS.resolve instead');
      },
  createFolder() {
        abort('FS.createFolder has been removed; use FS.mkdir instead');
      },
  createLink() {
        abort('FS.createLink has been removed; use FS.symlink instead');
      },
  joinPath() {
        abort('FS.joinPath has been removed; use PATH.join instead');
      },
  mmapAlloc() {
        abort('FS.mmapAlloc has been replaced by the top level function mmapAlloc');
      },
  standardizePath() {
        abort('FS.standardizePath has been removed; use PATH.normalize instead');
      },
  };
  
  var SYSCALLS = {
  DEFAULT_POLLMASK:5,
  calculateAt(dirfd, path, allowEmpty) {
        if (PATH.isAbs(path)) {
          return path;
        }
        // relative path
        var dir;
        if (dirfd === -100) {
          dir = FS.cwd();
        } else {
          var dirstream = SYSCALLS.getStreamFromFD(dirfd);
          dir = dirstream.path;
        }
        if (path.length == 0) {
          if (!allowEmpty) {
            throw new FS.ErrnoError(44);;
          }
          return dir;
        }
        return PATH.join2(dir, path);
      },
  doStat(func, path, buf) {
        var stat = func(path);
        HEAP32[((buf)>>2)] = stat.dev;
        HEAP32[(((buf)+(4))>>2)] = stat.mode;
        HEAPU32[(((buf)+(8))>>2)] = stat.nlink;
        HEAP32[(((buf)+(12))>>2)] = stat.uid;
        HEAP32[(((buf)+(16))>>2)] = stat.gid;
        HEAP32[(((buf)+(20))>>2)] = stat.rdev;
        (tempI64 = [stat.size>>>0,(tempDouble = stat.size,(+(Math.abs(tempDouble))) >= 1.0 ? (tempDouble > 0.0 ? (+(Math.floor((tempDouble)/4294967296.0)))>>>0 : (~~((+(Math.ceil((tempDouble - +(((~~(tempDouble)))>>>0))/4294967296.0)))))>>>0) : 0)], HEAP32[(((buf)+(24))>>2)] = tempI64[0],HEAP32[(((buf)+(28))>>2)] = tempI64[1]);
        HEAP32[(((buf)+(32))>>2)] = 4096;
        HEAP32[(((buf)+(36))>>2)] = stat.blocks;
        var atime = stat.atime.getTime();
        var mtime = stat.mtime.getTime();
        var ctime = stat.ctime.getTime();
        (tempI64 = [Math.floor(atime / 1000)>>>0,(tempDouble = Math.floor(atime / 1000),(+(Math.abs(tempDouble))) >= 1.0 ? (tempDouble > 0.0 ? (+(Math.floor((tempDouble)/4294967296.0)))>>>0 : (~~((+(Math.ceil((tempDouble - +(((~~(tempDouble)))>>>0))/4294967296.0)))))>>>0) : 0)], HEAP32[(((buf)+(40))>>2)] = tempI64[0],HEAP32[(((buf)+(44))>>2)] = tempI64[1]);
        HEAPU32[(((buf)+(48))>>2)] = (atime % 1000) * 1000 * 1000;
        (tempI64 = [Math.floor(mtime / 1000)>>>0,(tempDouble = Math.floor(mtime / 1000),(+(Math.abs(tempDouble))) >= 1.0 ? (tempDouble > 0.0 ? (+(Math.floor((tempDouble)/4294967296.0)))>>>0 : (~~((+(Math.ceil((tempDouble - +(((~~(tempDouble)))>>>0))/4294967296.0)))))>>>0) : 0)], HEAP32[(((buf)+(56))>>2)] = tempI64[0],HEAP32[(((buf)+(60))>>2)] = tempI64[1]);
        HEAPU32[(((buf)+(64))>>2)] = (mtime % 1000) * 1000 * 1000;
        (tempI64 = [Math.floor(ctime / 1000)>>>0,(tempDouble = Math.floor(ctime / 1000),(+(Math.abs(tempDouble))) >= 1.0 ? (tempDouble > 0.0 ? (+(Math.floor((tempDouble)/4294967296.0)))>>>0 : (~~((+(Math.ceil((tempDouble - +(((~~(tempDouble)))>>>0))/4294967296.0)))))>>>0) : 0)], HEAP32[(((buf)+(72))>>2)] = tempI64[0],HEAP32[(((buf)+(76))>>2)] = tempI64[1]);
        HEAPU32[(((buf)+(80))>>2)] = (ctime % 1000) * 1000 * 1000;
        (tempI64 = [stat.ino>>>0,(tempDouble = stat.ino,(+(Math.abs(tempDouble))) >= 1.0 ? (tempDouble > 0.0 ? (+(Math.floor((tempDouble)/4294967296.0)))>>>0 : (~~((+(Math.ceil((tempDouble - +(((~~(tempDouble)))>>>0))/4294967296.0)))))>>>0) : 0)], HEAP32[(((buf)+(88))>>2)] = tempI64[0],HEAP32[(((buf)+(92))>>2)] = tempI64[1]);
        return 0;
      },
  doMsync(addr, stream, len, flags, offset) {
        if (!FS.isFile(stream.node.mode)) {
          throw new FS.ErrnoError(43);
        }
        if (flags & 2) {
          // MAP_PRIVATE calls need not to be synced back to underlying fs
          return 0;
        }
        var buffer = HEAPU8.slice(addr, addr + len);
        FS.msync(stream, buffer, offset, len, flags);
      },
  getStreamFromFD(fd) {
        var stream = FS.getStreamChecked(fd);
        return stream;
      },
  varargs:undefined,
  getStr(ptr) {
        var ret = UTF8ToString(ptr);
        return ret;
      },
  };
  function ___syscall_fcntl64(fd, cmd, varargs) {
  SYSCALLS.varargs = varargs;
  try {
  
      var stream = SYSCALLS.getStreamFromFD(fd);
      switch (cmd) {
        case 0: {
          var arg = syscallGetVarargI();
          if (arg < 0) {
            return -28;
          }
          while (FS.streams[arg]) {
            arg++;
          }
          var newStream;
          newStream = FS.dupStream(stream, arg);
          return newStream.fd;
        }
        case 1:
        case 2:
          return 0;  // FD_CLOEXEC makes no sense for a single process.
        case 3:
          return stream.flags;
        case 4: {
          var arg = syscallGetVarargI();
          stream.flags |= arg;
          return 0;
        }
        case 12: {
          var arg = syscallGetVarargP();
          var offset = 0;
          // We're always unlocked.
          HEAP16[(((arg)+(offset))>>1)] = 2;
          return 0;
        }
        case 13:
        case 14:
          return 0; // Pretend that the locking is successful.
      }
      return -28;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  function ___syscall_fstat64(fd, buf) {
  try {
  
      var stream = SYSCALLS.getStreamFromFD(fd);
      return SYSCALLS.doStat(FS.stat, stream.path, buf);
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  var convertI32PairToI53Checked = (lo, hi) => {
      assert(lo == (lo >>> 0) || lo == (lo|0)); // lo should either be a i32 or a u32
      assert(hi === (hi|0));                    // hi should be a i32
      return ((hi + 0x200000) >>> 0 < 0x400001 - !!lo) ? (lo >>> 0) + hi * 4294967296 : NaN;
    };
  function ___syscall_ftruncate64(fd,length_low, length_high) {
    var length = convertI32PairToI53Checked(length_low, length_high);
  
    
  try {
  
      if (isNaN(length)) return 61;
      FS.ftruncate(fd, length);
      return 0;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  ;
  }

  var stringToUTF8 = (str, outPtr, maxBytesToWrite) => {
      assert(typeof maxBytesToWrite == 'number', 'stringToUTF8(str, outPtr, maxBytesToWrite) is missing the third parameter that specifies the length of the output buffer!');
      return stringToUTF8Array(str, HEAPU8, outPtr, maxBytesToWrite);
    };
  
  function ___syscall_getdents64(fd, dirp, count) {
  try {
  
      var stream = SYSCALLS.getStreamFromFD(fd)
      stream.getdents ||= FS.readdir(stream.path);
  
      var struct_size = 280;
      var pos = 0;
      var off = FS.llseek(stream, 0, 1);
  
      var idx = Math.floor(off / struct_size);
  
      while (idx < stream.getdents.length && pos + struct_size <= count) {
        var id;
        var type;
        var name = stream.getdents[idx];
        if (name === '.') {
          id = stream.node.id;
          type = 4; // DT_DIR
        }
        else if (name === '..') {
          var lookup = FS.lookupPath(stream.path, { parent: true });
          id = lookup.node.id;
          type = 4; // DT_DIR
        }
        else {
          var child = FS.lookupNode(stream.node, name);
          id = child.id;
          type = FS.isChrdev(child.mode) ? 2 :  // DT_CHR, character device.
                 FS.isDir(child.mode) ? 4 :     // DT_DIR, directory.
                 FS.isLink(child.mode) ? 10 :   // DT_LNK, symbolic link.
                 8;                             // DT_REG, regular file.
        }
        assert(id);
        (tempI64 = [id>>>0,(tempDouble = id,(+(Math.abs(tempDouble))) >= 1.0 ? (tempDouble > 0.0 ? (+(Math.floor((tempDouble)/4294967296.0)))>>>0 : (~~((+(Math.ceil((tempDouble - +(((~~(tempDouble)))>>>0))/4294967296.0)))))>>>0) : 0)], HEAP32[((dirp + pos)>>2)] = tempI64[0],HEAP32[(((dirp + pos)+(4))>>2)] = tempI64[1]);
        (tempI64 = [(idx + 1) * struct_size>>>0,(tempDouble = (idx + 1) * struct_size,(+(Math.abs(tempDouble))) >= 1.0 ? (tempDouble > 0.0 ? (+(Math.floor((tempDouble)/4294967296.0)))>>>0 : (~~((+(Math.ceil((tempDouble - +(((~~(tempDouble)))>>>0))/4294967296.0)))))>>>0) : 0)], HEAP32[(((dirp + pos)+(8))>>2)] = tempI64[0],HEAP32[(((dirp + pos)+(12))>>2)] = tempI64[1]);
        HEAP16[(((dirp + pos)+(16))>>1)] = 280;
        HEAP8[(dirp + pos)+(18)] = type;
        stringToUTF8(name, dirp + pos + 19, 256);
        pos += struct_size;
        idx += 1;
      }
      FS.llseek(stream, idx * struct_size, 0);
      return pos;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  
  function ___syscall_ioctl(fd, op, varargs) {
  SYSCALLS.varargs = varargs;
  try {
  
      var stream = SYSCALLS.getStreamFromFD(fd);
      switch (op) {
        case 21509: {
          if (!stream.tty) return -59;
          return 0;
        }
        case 21505: {
          if (!stream.tty) return -59;
          if (stream.tty.ops.ioctl_tcgets) {
            var termios = stream.tty.ops.ioctl_tcgets(stream);
            var argp = syscallGetVarargP();
            HEAP32[((argp)>>2)] = termios.c_iflag || 0;
            HEAP32[(((argp)+(4))>>2)] = termios.c_oflag || 0;
            HEAP32[(((argp)+(8))>>2)] = termios.c_cflag || 0;
            HEAP32[(((argp)+(12))>>2)] = termios.c_lflag || 0;
            for (var i = 0; i < 32; i++) {
              HEAP8[(argp + i)+(17)] = termios.c_cc[i] || 0;
            }
            return 0;
          }
          return 0;
        }
        case 21510:
        case 21511:
        case 21512: {
          if (!stream.tty) return -59;
          return 0; // no-op, not actually adjusting terminal settings
        }
        case 21506:
        case 21507:
        case 21508: {
          if (!stream.tty) return -59;
          if (stream.tty.ops.ioctl_tcsets) {
            var argp = syscallGetVarargP();
            var c_iflag = HEAP32[((argp)>>2)];
            var c_oflag = HEAP32[(((argp)+(4))>>2)];
            var c_cflag = HEAP32[(((argp)+(8))>>2)];
            var c_lflag = HEAP32[(((argp)+(12))>>2)];
            var c_cc = []
            for (var i = 0; i < 32; i++) {
              c_cc.push(HEAP8[(argp + i)+(17)]);
            }
            return stream.tty.ops.ioctl_tcsets(stream.tty, op, { c_iflag, c_oflag, c_cflag, c_lflag, c_cc });
          }
          return 0; // no-op, not actually adjusting terminal settings
        }
        case 21519: {
          if (!stream.tty) return -59;
          var argp = syscallGetVarargP();
          HEAP32[((argp)>>2)] = 0;
          return 0;
        }
        case 21520: {
          if (!stream.tty) return -59;
          return -28; // not supported
        }
        case 21531: {
          var argp = syscallGetVarargP();
          return FS.ioctl(stream, op, argp);
        }
        case 21523: {
          // TODO: in theory we should write to the winsize struct that gets
          // passed in, but for now musl doesn't read anything on it
          if (!stream.tty) return -59;
          if (stream.tty.ops.ioctl_tiocgwinsz) {
            var winsize = stream.tty.ops.ioctl_tiocgwinsz(stream.tty);
            var argp = syscallGetVarargP();
            HEAP16[((argp)>>1)] = winsize[0];
            HEAP16[(((argp)+(2))>>1)] = winsize[1];
          }
          return 0;
        }
        case 21524: {
          // TODO: technically, this ioctl call should change the window size.
          // but, since emscripten doesn't have any concept of a terminal window
          // yet, we'll just silently throw it away as we do TIOCGWINSZ
          if (!stream.tty) return -59;
          return 0;
        }
        case 21515: {
          if (!stream.tty) return -59;
          return 0;
        }
        default: return -28; // not supported
      }
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  function ___syscall_lstat64(path, buf) {
  try {
  
      path = SYSCALLS.getStr(path);
      return SYSCALLS.doStat(FS.lstat, path, buf);
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  function ___syscall_newfstatat(dirfd, path, buf, flags) {
  try {
  
      path = SYSCALLS.getStr(path);
      var nofollow = flags & 256;
      var allowEmpty = flags & 4096;
      flags = flags & (~6400);
      assert(!flags, `unknown flags in __syscall_newfstatat: ${flags}`);
      path = SYSCALLS.calculateAt(dirfd, path, allowEmpty);
      return SYSCALLS.doStat(nofollow ? FS.lstat : FS.stat, path, buf);
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  
  function ___syscall_openat(dirfd, path, flags, varargs) {
  SYSCALLS.varargs = varargs;
  try {
  
      path = SYSCALLS.getStr(path);
      path = SYSCALLS.calculateAt(dirfd, path);
      var mode = varargs ? syscallGetVarargI() : 0;
      return FS.open(path, flags, mode).fd;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  function ___syscall_rmdir(path) {
  try {
  
      path = SYSCALLS.getStr(path);
      FS.rmdir(path);
      return 0;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  function ___syscall_stat64(path, buf) {
  try {
  
      path = SYSCALLS.getStr(path);
      return SYSCALLS.doStat(FS.stat, path, buf);
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  function ___syscall_unlinkat(dirfd, path, flags) {
  try {
  
      path = SYSCALLS.getStr(path);
      path = SYSCALLS.calculateAt(dirfd, path);
      if (flags === 0) {
        FS.unlink(path);
      } else if (flags === 512) {
        FS.rmdir(path);
      } else {
        abort('Invalid flags passed to unlinkat');
      }
      return 0;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return -e.errno;
  }
  }

  var __abort_js = () => {
      abort('native code called abort()');
    };

  var __emscripten_memcpy_js = (dest, src, num) => HEAPU8.copyWithin(dest, src, src + num);

  var __emscripten_throw_longjmp = () => {
      throw Infinity;
    };

  function __gmtime_js(time_low, time_high,tmPtr) {
    var time = convertI32PairToI53Checked(time_low, time_high);
  
    
      var date = new Date(time * 1000);
      HEAP32[((tmPtr)>>2)] = date.getUTCSeconds();
      HEAP32[(((tmPtr)+(4))>>2)] = date.getUTCMinutes();
      HEAP32[(((tmPtr)+(8))>>2)] = date.getUTCHours();
      HEAP32[(((tmPtr)+(12))>>2)] = date.getUTCDate();
      HEAP32[(((tmPtr)+(16))>>2)] = date.getUTCMonth();
      HEAP32[(((tmPtr)+(20))>>2)] = date.getUTCFullYear()-1900;
      HEAP32[(((tmPtr)+(24))>>2)] = date.getUTCDay();
      var start = Date.UTC(date.getUTCFullYear(), 0, 1, 0, 0, 0, 0);
      var yday = ((date.getTime() - start) / (1000 * 60 * 60 * 24))|0;
      HEAP32[(((tmPtr)+(28))>>2)] = yday;
    ;
  }

  var isLeapYear = (year) => year%4 === 0 && (year%100 !== 0 || year%400 === 0);
  
  var MONTH_DAYS_LEAP_CUMULATIVE = [0,31,60,91,121,152,182,213,244,274,305,335];
  
  var MONTH_DAYS_REGULAR_CUMULATIVE = [0,31,59,90,120,151,181,212,243,273,304,334];
  var ydayFromDate = (date) => {
      var leap = isLeapYear(date.getFullYear());
      var monthDaysCumulative = (leap ? MONTH_DAYS_LEAP_CUMULATIVE : MONTH_DAYS_REGULAR_CUMULATIVE);
      var yday = monthDaysCumulative[date.getMonth()] + date.getDate() - 1; // -1 since it's days since Jan 1
  
      return yday;
    };
  
  function __localtime_js(time_low, time_high,tmPtr) {
    var time = convertI32PairToI53Checked(time_low, time_high);
  
    
      var date = new Date(time*1000);
      HEAP32[((tmPtr)>>2)] = date.getSeconds();
      HEAP32[(((tmPtr)+(4))>>2)] = date.getMinutes();
      HEAP32[(((tmPtr)+(8))>>2)] = date.getHours();
      HEAP32[(((tmPtr)+(12))>>2)] = date.getDate();
      HEAP32[(((tmPtr)+(16))>>2)] = date.getMonth();
      HEAP32[(((tmPtr)+(20))>>2)] = date.getFullYear()-1900;
      HEAP32[(((tmPtr)+(24))>>2)] = date.getDay();
  
      var yday = ydayFromDate(date)|0;
      HEAP32[(((tmPtr)+(28))>>2)] = yday;
      HEAP32[(((tmPtr)+(36))>>2)] = -(date.getTimezoneOffset() * 60);
  
      // Attention: DST is in December in South, and some regions don't have DST at all.
      var start = new Date(date.getFullYear(), 0, 1);
      var summerOffset = new Date(date.getFullYear(), 6, 1).getTimezoneOffset();
      var winterOffset = start.getTimezoneOffset();
      var dst = (summerOffset != winterOffset && date.getTimezoneOffset() == Math.min(winterOffset, summerOffset))|0;
      HEAP32[(((tmPtr)+(32))>>2)] = dst;
    ;
  }

  
  var __tzset_js = (timezone, daylight, std_name, dst_name) => {
      // TODO: Use (malleable) environment variables instead of system settings.
      var currentYear = new Date().getFullYear();
      var winter = new Date(currentYear, 0, 1);
      var summer = new Date(currentYear, 6, 1);
      var winterOffset = winter.getTimezoneOffset();
      var summerOffset = summer.getTimezoneOffset();
  
      // Local standard timezone offset. Local standard time is not adjusted for
      // daylight savings.  This code uses the fact that getTimezoneOffset returns
      // a greater value during Standard Time versus Daylight Saving Time (DST).
      // Thus it determines the expected output during Standard Time, and it
      // compares whether the output of the given date the same (Standard) or less
      // (DST).
      var stdTimezoneOffset = Math.max(winterOffset, summerOffset);
  
      // timezone is specified as seconds west of UTC ("The external variable
      // `timezone` shall be set to the difference, in seconds, between
      // Coordinated Universal Time (UTC) and local standard time."), the same
      // as returned by stdTimezoneOffset.
      // See http://pubs.opengroup.org/onlinepubs/009695399/functions/tzset.html
      HEAPU32[((timezone)>>2)] = stdTimezoneOffset * 60;
  
      HEAP32[((daylight)>>2)] = Number(winterOffset != summerOffset);
  
      var extractZone = (timezoneOffset) => {
        // Why inverse sign?
        // Read here https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTimezoneOffset
        var sign = timezoneOffset >= 0 ? "-" : "+";
  
        var absOffset = Math.abs(timezoneOffset)
        var hours = String(Math.floor(absOffset / 60)).padStart(2, "0");
        var minutes = String(absOffset % 60).padStart(2, "0");
  
        return `UTC${sign}${hours}${minutes}`;
      }
  
      var winterName = extractZone(winterOffset);
      var summerName = extractZone(summerOffset);
      assert(winterName);
      assert(summerName);
      assert(lengthBytesUTF8(winterName) <= 16, `timezone name truncated to fit in TZNAME_MAX (${winterName})`);
      assert(lengthBytesUTF8(summerName) <= 16, `timezone name truncated to fit in TZNAME_MAX (${summerName})`);
      if (summerOffset < winterOffset) {
        // Northern hemisphere
        stringToUTF8(winterName, std_name, 17);
        stringToUTF8(summerName, dst_name, 17);
      } else {
        stringToUTF8(winterName, dst_name, 17);
        stringToUTF8(summerName, std_name, 17);
      }
    };

  var _emscripten_date_now = () => Date.now();

  var getHeapMax = () =>
      // Stay one Wasm page short of 4GB: while e.g. Chrome is able to allocate
      // full 4GB Wasm memories, the size will wrap back to 0 bytes in Wasm side
      // for any code that deals with heap sizes, which would require special
      // casing all heap size related code to treat 0 specially.
      2147483648;
  
  
  var growMemory = (size) => {
      var b = wasmMemory.buffer;
      var pages = ((size - b.byteLength + 65535) / 65536) | 0;
      try {
        // round size grow request up to wasm page size (fixed 64KB per spec)
        wasmMemory.grow(pages); // .grow() takes a delta compared to the previous size
        updateMemoryViews();
        return 1 /*success*/;
      } catch(e) {
        err(`growMemory: Attempted to grow heap from ${b.byteLength} bytes to ${size} bytes, but got error: ${e}`);
      }
      // implicit 0 return to save code size (caller will cast "undefined" into 0
      // anyhow)
    };
  var _emscripten_resize_heap = (requestedSize) => {
      var oldSize = HEAPU8.length;
      // With CAN_ADDRESS_2GB or MEMORY64, pointers are already unsigned.
      requestedSize >>>= 0;
      // With multithreaded builds, races can happen (another thread might increase the size
      // in between), so return a failure, and let the caller retry.
      assert(requestedSize > oldSize);
  
      // Memory resize rules:
      // 1.  Always increase heap size to at least the requested size, rounded up
      //     to next page multiple.
      // 2a. If MEMORY_GROWTH_LINEAR_STEP == -1, excessively resize the heap
      //     geometrically: increase the heap size according to
      //     MEMORY_GROWTH_GEOMETRIC_STEP factor (default +20%), At most
      //     overreserve by MEMORY_GROWTH_GEOMETRIC_CAP bytes (default 96MB).
      // 2b. If MEMORY_GROWTH_LINEAR_STEP != -1, excessively resize the heap
      //     linearly: increase the heap size by at least
      //     MEMORY_GROWTH_LINEAR_STEP bytes.
      // 3.  Max size for the heap is capped at 2048MB-WASM_PAGE_SIZE, or by
      //     MAXIMUM_MEMORY, or by ASAN limit, depending on which is smallest
      // 4.  If we were unable to allocate as much memory, it may be due to
      //     over-eager decision to excessively reserve due to (3) above.
      //     Hence if an allocation fails, cut down on the amount of excess
      //     growth, in an attempt to succeed to perform a smaller allocation.
  
      // A limit is set for how much we can grow. We should not exceed that
      // (the wasm binary specifies it, so if we tried, we'd fail anyhow).
      var maxHeapSize = getHeapMax();
      if (requestedSize > maxHeapSize) {
        err(`Cannot enlarge memory, requested ${requestedSize} bytes, but the limit is ${maxHeapSize} bytes!`);
        return false;
      }
  
      // Loop through potential heap size increases. If we attempt a too eager
      // reservation that fails, cut down on the attempted size and reserve a
      // smaller bump instead. (max 3 times, chosen somewhat arbitrarily)
      for (var cutDown = 1; cutDown <= 4; cutDown *= 2) {
        var overGrownHeapSize = oldSize * (1 + 0.2 / cutDown); // ensure geometric growth
        // but limit overreserving (default to capping at +96MB overgrowth at most)
        overGrownHeapSize = Math.min(overGrownHeapSize, requestedSize + 100663296 );
  
        var newSize = Math.min(maxHeapSize, alignMemory(Math.max(requestedSize, overGrownHeapSize), 65536));
  
        var replacement = growMemory(newSize);
        if (replacement) {
  
          return true;
        }
      }
      err(`Failed to grow the heap from ${oldSize} bytes to ${newSize} bytes, not enough memory!`);
      return false;
    };

  var ENV = {
  };
  
  var getExecutableName = () => {
      return thisProgram || './this.program';
    };
  var getEnvStrings = () => {
      if (!getEnvStrings.strings) {
        // Default values.
        // Browser language detection #8751
        var lang = ((typeof navigator == 'object' && navigator.languages && navigator.languages[0]) || 'C').replace('-', '_') + '.UTF-8';
        var env = {
          'USER': 'web_user',
          'LOGNAME': 'web_user',
          'PATH': '/',
          'PWD': '/',
          'HOME': '/home/web_user',
          'LANG': lang,
          '_': getExecutableName()
        };
        // Apply the user-provided values, if any.
        for (var x in ENV) {
          // x is a key in ENV; if ENV[x] is undefined, that means it was
          // explicitly set to be so. We allow user code to do that to
          // force variables with default values to remain unset.
          if (ENV[x] === undefined) delete env[x];
          else env[x] = ENV[x];
        }
        var strings = [];
        for (var x in env) {
          strings.push(`${x}=${env[x]}`);
        }
        getEnvStrings.strings = strings;
      }
      return getEnvStrings.strings;
    };
  
  var stringToAscii = (str, buffer) => {
      for (var i = 0; i < str.length; ++i) {
        assert(str.charCodeAt(i) === (str.charCodeAt(i) & 0xff));
        HEAP8[buffer++] = str.charCodeAt(i);
      }
      // Null-terminate the string
      HEAP8[buffer] = 0;
    };
  var _environ_get = (__environ, environ_buf) => {
      var bufSize = 0;
      getEnvStrings().forEach((string, i) => {
        var ptr = environ_buf + bufSize;
        HEAPU32[(((__environ)+(i*4))>>2)] = ptr;
        stringToAscii(string, ptr);
        bufSize += string.length + 1;
      });
      return 0;
    };

  var _environ_sizes_get = (penviron_count, penviron_buf_size) => {
      var strings = getEnvStrings();
      HEAPU32[((penviron_count)>>2)] = strings.length;
      var bufSize = 0;
      strings.forEach((string) => bufSize += string.length + 1);
      HEAPU32[((penviron_buf_size)>>2)] = bufSize;
      return 0;
    };

  function _fd_close(fd) {
  try {
  
      var stream = SYSCALLS.getStreamFromFD(fd);
      FS.close(stream);
      return 0;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return e.errno;
  }
  }

  /** @param {number=} offset */
  var doReadv = (stream, iov, iovcnt, offset) => {
      var ret = 0;
      for (var i = 0; i < iovcnt; i++) {
        var ptr = HEAPU32[((iov)>>2)];
        var len = HEAPU32[(((iov)+(4))>>2)];
        iov += 8;
        var curr = FS.read(stream, HEAP8, ptr, len, offset);
        if (curr < 0) return -1;
        ret += curr;
        if (curr < len) break; // nothing more to read
        if (typeof offset != 'undefined') {
          offset += curr;
        }
      }
      return ret;
    };
  
  function _fd_read(fd, iov, iovcnt, pnum) {
  try {
  
      var stream = SYSCALLS.getStreamFromFD(fd);
      var num = doReadv(stream, iov, iovcnt);
      HEAPU32[((pnum)>>2)] = num;
      return 0;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return e.errno;
  }
  }

  
  function _fd_seek(fd,offset_low, offset_high,whence,newOffset) {
    var offset = convertI32PairToI53Checked(offset_low, offset_high);
  
    
  try {
  
      if (isNaN(offset)) return 61;
      var stream = SYSCALLS.getStreamFromFD(fd);
      FS.llseek(stream, offset, whence);
      (tempI64 = [stream.position>>>0,(tempDouble = stream.position,(+(Math.abs(tempDouble))) >= 1.0 ? (tempDouble > 0.0 ? (+(Math.floor((tempDouble)/4294967296.0)))>>>0 : (~~((+(Math.ceil((tempDouble - +(((~~(tempDouble)))>>>0))/4294967296.0)))))>>>0) : 0)], HEAP32[((newOffset)>>2)] = tempI64[0],HEAP32[(((newOffset)+(4))>>2)] = tempI64[1]);
      if (stream.getdents && offset === 0 && whence === 0) stream.getdents = null; // reset readdir state
      return 0;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return e.errno;
  }
  ;
  }

  function _fd_sync(fd) {
  try {
  
      var stream = SYSCALLS.getStreamFromFD(fd);
      if (stream.stream_ops?.fsync) {
        return stream.stream_ops.fsync(stream);
      }
      return 0; // we can't do anything synchronously; the in-memory FS is already synced to
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return e.errno;
  }
  }

  /** @param {number=} offset */
  var doWritev = (stream, iov, iovcnt, offset) => {
      var ret = 0;
      for (var i = 0; i < iovcnt; i++) {
        var ptr = HEAPU32[((iov)>>2)];
        var len = HEAPU32[(((iov)+(4))>>2)];
        iov += 8;
        var curr = FS.write(stream, HEAP8, ptr, len, offset);
        if (curr < 0) return -1;
        ret += curr;
        if (curr < len) {
          // No more space to write.
          break;
        }
        if (typeof offset != 'undefined') {
          offset += curr;
        }
      }
      return ret;
    };
  
  function _fd_write(fd, iov, iovcnt, pnum) {
  try {
  
      var stream = SYSCALLS.getStreamFromFD(fd);
      var num = doWritev(stream, iov, iovcnt);
      HEAPU32[((pnum)>>2)] = num;
      return 0;
    } catch (e) {
    if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
    return e.errno;
  }
  }

  var wasmTableMirror = [];
  
  /** @type {WebAssembly.Table} */
  var wasmTable;
  var getWasmTableEntry = (funcPtr) => {
      var func = wasmTableMirror[funcPtr];
      if (!func) {
        if (funcPtr >= wasmTableMirror.length) wasmTableMirror.length = funcPtr + 1;
        wasmTableMirror[funcPtr] = func = wasmTable.get(funcPtr);
      }
      assert(wasmTable.get(funcPtr) == func, 'JavaScript-side Wasm function table mirror is out of date!');
      return func;
    };

  var UTF16Decoder = typeof TextDecoder != 'undefined' ? new TextDecoder('utf-16le') : undefined;;
  var UTF16ToString = (ptr, maxBytesToRead) => {
      assert(ptr % 2 == 0, 'Pointer passed to UTF16ToString must be aligned to two bytes!');
      var endPtr = ptr;
      // TextDecoder needs to know the byte length in advance, it doesn't stop on
      // null terminator by itself.
      // Also, use the length info to avoid running tiny strings through
      // TextDecoder, since .subarray() allocates garbage.
      var idx = endPtr >> 1;
      var maxIdx = idx + maxBytesToRead / 2;
      // If maxBytesToRead is not passed explicitly, it will be undefined, and this
      // will always evaluate to true. This saves on code size.
      while (!(idx >= maxIdx) && HEAPU16[idx]) ++idx;
      endPtr = idx << 1;
  
      if (endPtr - ptr > 32 && UTF16Decoder)
        return UTF16Decoder.decode(HEAPU8.subarray(ptr, endPtr));
  
      // Fallback: decode without UTF16Decoder
      var str = '';
  
      // If maxBytesToRead is not passed explicitly, it will be undefined, and the
      // for-loop's condition will always evaluate to true. The loop is then
      // terminated on the first null char.
      for (var i = 0; !(i >= maxBytesToRead / 2); ++i) {
        var codeUnit = HEAP16[(((ptr)+(i*2))>>1)];
        if (codeUnit == 0) break;
        // fromCharCode constructs a character from a UTF-16 code unit, so we can
        // pass the UTF16 string right through.
        str += String.fromCharCode(codeUnit);
      }
  
      return str;
    };


  var uleb128Encode = (n, target) => {
      assert(n < 16384);
      if (n < 128) {
        target.push(n);
      } else {
        target.push((n % 128) | 128, n >> 7);
      }
    };
  
  var sigToWasmTypes = (sig) => {
      assert(!sig.includes('j'), 'i64 not permitted in function signatures when WASM_BIGINT is disabled');
      var typeNames = {
        'i': 'i32',
        'j': 'i64',
        'f': 'f32',
        'd': 'f64',
        'e': 'externref',
        'p': 'i32',
      };
      var type = {
        parameters: [],
        results: sig[0] == 'v' ? [] : [typeNames[sig[0]]]
      };
      for (var i = 1; i < sig.length; ++i) {
        assert(sig[i] in typeNames, 'invalid signature char: ' + sig[i]);
        type.parameters.push(typeNames[sig[i]]);
      }
      return type;
    };
  
  var generateFuncType = (sig, target) => {
      var sigRet = sig.slice(0, 1);
      var sigParam = sig.slice(1);
      var typeCodes = {
        'i': 0x7f, // i32
        'p': 0x7f, // i32
        'j': 0x7e, // i64
        'f': 0x7d, // f32
        'd': 0x7c, // f64
        'e': 0x6f, // externref
      };
  
      // Parameters, length + signatures
      target.push(0x60 /* form: func */);
      uleb128Encode(sigParam.length, target);
      for (var i = 0; i < sigParam.length; ++i) {
        assert(sigParam[i] in typeCodes, 'invalid signature char: ' + sigParam[i]);
        target.push(typeCodes[sigParam[i]]);
      }
  
      // Return values, length + signatures
      // With no multi-return in MVP, either 0 (void) or 1 (anything else)
      if (sigRet == 'v') {
        target.push(0x00);
      } else {
        target.push(0x01, typeCodes[sigRet]);
      }
    };
  var convertJsFunctionToWasm = (func, sig) => {
  
      assert(!sig.includes('j'), 'i64 not permitted in function signatures when WASM_BIGINT is disabled');
  
      // If the type reflection proposal is available, use the new
      // "WebAssembly.Function" constructor.
      // Otherwise, construct a minimal wasm module importing the JS function and
      // re-exporting it.
      if (typeof WebAssembly.Function == "function") {
        return new WebAssembly.Function(sigToWasmTypes(sig), func);
      }
  
      // The module is static, with the exception of the type section, which is
      // generated based on the signature passed in.
      var typeSectionBody = [
        0x01, // count: 1
      ];
      generateFuncType(sig, typeSectionBody);
  
      // Rest of the module is static
      var bytes = [
        0x00, 0x61, 0x73, 0x6d, // magic ("\0asm")
        0x01, 0x00, 0x00, 0x00, // version: 1
        0x01, // Type section code
      ];
      // Write the overall length of the type section followed by the body
      uleb128Encode(typeSectionBody.length, bytes);
      bytes.push(...typeSectionBody);
  
      // The rest of the module is static
      bytes.push(
        0x02, 0x07, // import section
          // (import "e" "f" (func 0 (type 0)))
          0x01, 0x01, 0x65, 0x01, 0x66, 0x00, 0x00,
        0x07, 0x05, // export section
          // (export "f" (func 0 (type 0)))
          0x01, 0x01, 0x66, 0x00, 0x00,
      );
  
      // We can compile this wasm module synchronously because it is very small.
      // This accepts an import (at "e.f"), that it reroutes to an export (at "f")
      var module = new WebAssembly.Module(new Uint8Array(bytes));
      var instance = new WebAssembly.Instance(module, { 'e': { 'f': func } });
      var wrappedFunc = instance.exports['f'];
      return wrappedFunc;
    };
  
  
  var updateTableMap = (offset, count) => {
      if (functionsInTableMap) {
        for (var i = offset; i < offset + count; i++) {
          var item = getWasmTableEntry(i);
          // Ignore null values.
          if (item) {
            functionsInTableMap.set(item, i);
          }
        }
      }
    };
  
  var functionsInTableMap;
  
  var getFunctionAddress = (func) => {
      // First, create the map if this is the first use.
      if (!functionsInTableMap) {
        functionsInTableMap = new WeakMap();
        updateTableMap(0, wasmTable.length);
      }
      return functionsInTableMap.get(func) || 0;
    };
  
  
  var freeTableIndexes = [];
  
  var getEmptyTableSlot = () => {
      // Reuse a free index if there is one, otherwise grow.
      if (freeTableIndexes.length) {
        return freeTableIndexes.pop();
      }
      // Grow the table
      try {
        wasmTable.grow(1);
      } catch (err) {
        if (!(err instanceof RangeError)) {
          throw err;
        }
        throw 'Unable to grow wasm table. Set ALLOW_TABLE_GROWTH.';
      }
      return wasmTable.length - 1;
    };
  
  
  
  var setWasmTableEntry = (idx, func) => {
      wasmTable.set(idx, func);
      // With ABORT_ON_WASM_EXCEPTIONS wasmTable.get is overridden to return wrapped
      // functions so we need to call it here to retrieve the potential wrapper correctly
      // instead of just storing 'func' directly into wasmTableMirror
      wasmTableMirror[idx] = wasmTable.get(idx);
    };
  
  /** @param {string=} sig */
  var addFunction = (func, sig) => {
      assert(typeof func != 'undefined');
      // Check if the function is already in the table, to ensure each function
      // gets a unique index.
      var rtn = getFunctionAddress(func);
      if (rtn) {
        return rtn;
      }
  
      // It's not in the table, add it now.
  
      var ret = getEmptyTableSlot();
  
      // Set the new value.
      try {
        // Attempting to call this with JS function will cause of table.set() to fail
        setWasmTableEntry(ret, func);
      } catch (err) {
        if (!(err instanceof TypeError)) {
          throw err;
        }
        assert(typeof sig != 'undefined', 'Missing signature argument to addFunction: ' + func);
        var wrapped = convertJsFunctionToWasm(func, sig);
        setWasmTableEntry(ret, wrapped);
      }
  
      functionsInTableMap.set(func, ret);
  
      return ret;
    };

  var getCFunc = (ident) => {
      var func = Module['_' + ident]; // closure exported function
      assert(func, 'Cannot call unknown function ' + ident + ', make sure it is exported');
      return func;
    };
  
  var writeArrayToMemory = (array, buffer) => {
      assert(array.length >= 0, 'writeArrayToMemory array must have a length (should be an array or typed array)')
      HEAP8.set(array, buffer);
    };
  
  
  
  var stackAlloc = (sz) => __emscripten_stack_alloc(sz);
  var stringToUTF8OnStack = (str) => {
      var size = lengthBytesUTF8(str) + 1;
      var ret = stackAlloc(size);
      stringToUTF8(str, ret, size);
      return ret;
    };
  
  
  
  
  
    /**
     * @param {string|null=} returnType
     * @param {Array=} argTypes
     * @param {Arguments|Array=} args
     * @param {Object=} opts
     */
  var ccall = (ident, returnType, argTypes, args, opts) => {
      // For fast lookup of conversion functions
      var toC = {
        'string': (str) => {
          var ret = 0;
          if (str !== null && str !== undefined && str !== 0) { // null string
            ret = stringToUTF8OnStack(str);
          }
          return ret;
        },
        'array': (arr) => {
          var ret = stackAlloc(arr.length);
          writeArrayToMemory(arr, ret);
          return ret;
        }
      };
  
      function convertReturnValue(ret) {
        if (returnType === 'string') {
          return UTF8ToString(ret);
        }
        if (returnType === 'boolean') return Boolean(ret);
        return ret;
      }
  
      var func = getCFunc(ident);
      var cArgs = [];
      var stack = 0;
      assert(returnType !== 'array', 'Return type should not be "array".');
      if (args) {
        for (var i = 0; i < args.length; i++) {
          var converter = toC[argTypes[i]];
          if (converter) {
            if (stack === 0) stack = stackSave();
            cArgs[i] = converter(args[i]);
          } else {
            cArgs[i] = args[i];
          }
        }
      }
      var ret = func(...cArgs);
      function onDone(ret) {
        if (stack !== 0) stackRestore(stack);
        return convertReturnValue(ret);
      }
  
      ret = onDone(ret);
      return ret;
    };

  
  
    /**
     * @param {string=} returnType
     * @param {Array=} argTypes
     * @param {Object=} opts
     */
  var cwrap = (ident, returnType, argTypes, opts) => {
      return (...args) => ccall(ident, returnType, argTypes, args, opts);
    };


  
  
  
  
  var removeFunction = (index) => {
      functionsInTableMap.delete(getWasmTableEntry(index));
      setWasmTableEntry(index, null);
      freeTableIndexes.push(index);
    };


  var stringToUTF16 = (str, outPtr, maxBytesToWrite) => {
      assert(outPtr % 2 == 0, 'Pointer passed to stringToUTF16 must be aligned to two bytes!');
      assert(typeof maxBytesToWrite == 'number', 'stringToUTF16(str, outPtr, maxBytesToWrite) is missing the third parameter that specifies the length of the output buffer!');
      // Backwards compatibility: if max bytes is not specified, assume unsafe unbounded write is allowed.
      maxBytesToWrite ??= 0x7FFFFFFF;
      if (maxBytesToWrite < 2) return 0;
      maxBytesToWrite -= 2; // Null terminator.
      var startPtr = outPtr;
      var numCharsToWrite = (maxBytesToWrite < str.length*2) ? (maxBytesToWrite / 2) : str.length;
      for (var i = 0; i < numCharsToWrite; ++i) {
        // charCodeAt returns a UTF-16 encoded code unit, so it can be directly written to the HEAP.
        var codeUnit = str.charCodeAt(i); // possibly a lead surrogate
        HEAP16[((outPtr)>>1)] = codeUnit;
        outPtr += 2;
      }
      // Null-terminate the pointer to the HEAP.
      HEAP16[((outPtr)>>1)] = 0;
      return outPtr - startPtr;
    };


  FS.createPreloadedFile = FS_createPreloadedFile;
  FS.staticInit();
  // Set module methods based on EXPORTED_RUNTIME_METHODS
  ;
function checkIncomingModuleAPI() {
  ignoredModuleProp('fetchSettings');
}
var wasmImports = {
  /** @export */
  __assert_fail: ___assert_fail,
  /** @export */
  __syscall_fcntl64: ___syscall_fcntl64,
  /** @export */
  __syscall_fstat64: ___syscall_fstat64,
  /** @export */
  __syscall_ftruncate64: ___syscall_ftruncate64,
  /** @export */
  __syscall_getdents64: ___syscall_getdents64,
  /** @export */
  __syscall_ioctl: ___syscall_ioctl,
  /** @export */
  __syscall_lstat64: ___syscall_lstat64,
  /** @export */
  __syscall_newfstatat: ___syscall_newfstatat,
  /** @export */
  __syscall_openat: ___syscall_openat,
  /** @export */
  __syscall_rmdir: ___syscall_rmdir,
  /** @export */
  __syscall_stat64: ___syscall_stat64,
  /** @export */
  __syscall_unlinkat: ___syscall_unlinkat,
  /** @export */
  _abort_js: __abort_js,
  /** @export */
  _emscripten_memcpy_js: __emscripten_memcpy_js,
  /** @export */
  _emscripten_throw_longjmp: __emscripten_throw_longjmp,
  /** @export */
  _gmtime_js: __gmtime_js,
  /** @export */
  _localtime_js: __localtime_js,
  /** @export */
  _tzset_js: __tzset_js,
  /** @export */
  emscripten_date_now: _emscripten_date_now,
  /** @export */
  emscripten_resize_heap: _emscripten_resize_heap,
  /** @export */
  environ_get: _environ_get,
  /** @export */
  environ_sizes_get: _environ_sizes_get,
  /** @export */
  fd_close: _fd_close,
  /** @export */
  fd_read: _fd_read,
  /** @export */
  fd_seek: _fd_seek,
  /** @export */
  fd_sync: _fd_sync,
  /** @export */
  fd_write: _fd_write,
  /** @export */
  invoke_ii,
  /** @export */
  invoke_iii,
  /** @export */
  invoke_iiii,
  /** @export */
  invoke_iiiii,
  /** @export */
  invoke_v,
  /** @export */
  invoke_vii,
  /** @export */
  invoke_viii,
  /** @export */
  invoke_viiii,
  /** @export */
  invoke_viiiiiiiii
};
var wasmExports = createWasm();
var ___wasm_call_ctors = createExportWrapper('__wasm_call_ctors', 0);
var _PDFiumExt_Init = Module['_PDFiumExt_Init'] = createExportWrapper('PDFiumExt_Init', 0);
var _FPDF_InitLibraryWithConfig = Module['_FPDF_InitLibraryWithConfig'] = createExportWrapper('FPDF_InitLibraryWithConfig', 1);
var _PDFiumExt_OpenFileWriter = Module['_PDFiumExt_OpenFileWriter'] = createExportWrapper('PDFiumExt_OpenFileWriter', 0);
var _PDFiumExt_GetFileWriterSize = Module['_PDFiumExt_GetFileWriterSize'] = createExportWrapper('PDFiumExt_GetFileWriterSize', 1);
var _PDFiumExt_GetFileWriterData = Module['_PDFiumExt_GetFileWriterData'] = createExportWrapper('PDFiumExt_GetFileWriterData', 3);
var _PDFiumExt_CloseFileWriter = Module['_PDFiumExt_CloseFileWriter'] = createExportWrapper('PDFiumExt_CloseFileWriter', 1);
var _PDFiumExt_SaveAsCopy = Module['_PDFiumExt_SaveAsCopy'] = createExportWrapper('PDFiumExt_SaveAsCopy', 2);
var _FPDF_SaveAsCopy = Module['_FPDF_SaveAsCopy'] = createExportWrapper('FPDF_SaveAsCopy', 3);
var _PDFiumExt_OpenFormFillInfo = Module['_PDFiumExt_OpenFormFillInfo'] = createExportWrapper('PDFiumExt_OpenFormFillInfo', 0);
var _PDFiumExt_CloseFormFillInfo = Module['_PDFiumExt_CloseFormFillInfo'] = createExportWrapper('PDFiumExt_CloseFormFillInfo', 1);
var _PDFiumExt_InitFormFillEnvironment = Module['_PDFiumExt_InitFormFillEnvironment'] = createExportWrapper('PDFiumExt_InitFormFillEnvironment', 2);
var _FPDFDOC_InitFormFillEnvironment = Module['_FPDFDOC_InitFormFillEnvironment'] = createExportWrapper('FPDFDOC_InitFormFillEnvironment', 2);
var _PDFiumExt_ExitFormFillEnvironment = Module['_PDFiumExt_ExitFormFillEnvironment'] = createExportWrapper('PDFiumExt_ExitFormFillEnvironment', 1);
var _FPDFDOC_ExitFormFillEnvironment = Module['_FPDFDOC_ExitFormFillEnvironment'] = createExportWrapper('FPDFDOC_ExitFormFillEnvironment', 1);
var _EPDFNamedDest_SetDest = Module['_EPDFNamedDest_SetDest'] = createExportWrapper('EPDFNamedDest_SetDest', 3);
var _EPDFNamedDest_Remove = Module['_EPDFNamedDest_Remove'] = createExportWrapper('EPDFNamedDest_Remove', 2);
var _EPDFDest_CreateView = Module['_EPDFDest_CreateView'] = createExportWrapper('EPDFDest_CreateView', 4);
var _EPDFDest_CreateXYZ = Module['_EPDFDest_CreateXYZ'] = createExportWrapper('EPDFDest_CreateXYZ', 7);
var _EPDFDest_CreateRemoteView = Module['_EPDFDest_CreateRemoteView'] = createExportWrapper('EPDFDest_CreateRemoteView', 5);
var _EPDFDest_CreateRemoteXYZ = Module['_EPDFDest_CreateRemoteXYZ'] = createExportWrapper('EPDFDest_CreateRemoteXYZ', 8);
var _EPDFAction_CreateGoTo = Module['_EPDFAction_CreateGoTo'] = createExportWrapper('EPDFAction_CreateGoTo', 2);
var _EPDFAction_CreateGoToNamed = Module['_EPDFAction_CreateGoToNamed'] = createExportWrapper('EPDFAction_CreateGoToNamed', 2);
var _EPDFAction_CreateLaunch = Module['_EPDFAction_CreateLaunch'] = createExportWrapper('EPDFAction_CreateLaunch', 2);
var _EPDFAction_CreateRemoteGoToByName = Module['_EPDFAction_CreateRemoteGoToByName'] = createExportWrapper('EPDFAction_CreateRemoteGoToByName', 3);
var _EPDFAction_CreateRemoteGoToDest = Module['_EPDFAction_CreateRemoteGoToDest'] = createExportWrapper('EPDFAction_CreateRemoteGoToDest', 3);
var _EPDFAction_CreateURI = Module['_EPDFAction_CreateURI'] = createExportWrapper('EPDFAction_CreateURI', 2);
var _EPDFBookmark_Create = Module['_EPDFBookmark_Create'] = createExportWrapper('EPDFBookmark_Create', 2);
var _EPDFBookmark_Delete = Module['_EPDFBookmark_Delete'] = createExportWrapper('EPDFBookmark_Delete', 2);
var _EPDFBookmark_AppendChild = Module['_EPDFBookmark_AppendChild'] = createExportWrapper('EPDFBookmark_AppendChild', 3);
var _EPDFBookmark_InsertAfter = Module['_EPDFBookmark_InsertAfter'] = createExportWrapper('EPDFBookmark_InsertAfter', 4);
var _EPDFBookmark_Clear = Module['_EPDFBookmark_Clear'] = createExportWrapper('EPDFBookmark_Clear', 1);
var _EPDFBookmark_SetTitle = Module['_EPDFBookmark_SetTitle'] = createExportWrapper('EPDFBookmark_SetTitle', 2);
var _EPDFBookmark_SetDest = Module['_EPDFBookmark_SetDest'] = createExportWrapper('EPDFBookmark_SetDest', 3);
var _EPDFBookmark_SetAction = Module['_EPDFBookmark_SetAction'] = createExportWrapper('EPDFBookmark_SetAction', 3);
var _EPDFBookmark_ClearTarget = Module['_EPDFBookmark_ClearTarget'] = createExportWrapper('EPDFBookmark_ClearTarget', 1);
var _EPDF_PNG_EncodeRGBA = Module['_EPDF_PNG_EncodeRGBA'] = createExportWrapper('EPDF_PNG_EncodeRGBA', 6);
var _FPDFAnnot_IsSupportedSubtype = Module['_FPDFAnnot_IsSupportedSubtype'] = createExportWrapper('FPDFAnnot_IsSupportedSubtype', 1);
var _FPDFPage_CreateAnnot = Module['_FPDFPage_CreateAnnot'] = createExportWrapper('FPDFPage_CreateAnnot', 2);
var _FPDFPage_GetAnnotCount = Module['_FPDFPage_GetAnnotCount'] = createExportWrapper('FPDFPage_GetAnnotCount', 1);
var _FPDFPage_GetAnnot = Module['_FPDFPage_GetAnnot'] = createExportWrapper('FPDFPage_GetAnnot', 2);
var _FPDFPage_GetAnnotIndex = Module['_FPDFPage_GetAnnotIndex'] = createExportWrapper('FPDFPage_GetAnnotIndex', 2);
var _FPDFPage_CloseAnnot = Module['_FPDFPage_CloseAnnot'] = createExportWrapper('FPDFPage_CloseAnnot', 1);
var _FPDFPage_RemoveAnnot = Module['_FPDFPage_RemoveAnnot'] = createExportWrapper('FPDFPage_RemoveAnnot', 2);
var _FPDFAnnot_GetSubtype = Module['_FPDFAnnot_GetSubtype'] = createExportWrapper('FPDFAnnot_GetSubtype', 1);
var _FPDFAnnot_IsObjectSupportedSubtype = Module['_FPDFAnnot_IsObjectSupportedSubtype'] = createExportWrapper('FPDFAnnot_IsObjectSupportedSubtype', 1);
var _FPDFAnnot_UpdateObject = Module['_FPDFAnnot_UpdateObject'] = createExportWrapper('FPDFAnnot_UpdateObject', 2);
var _FPDFAnnot_AddInkStroke = Module['_FPDFAnnot_AddInkStroke'] = createExportWrapper('FPDFAnnot_AddInkStroke', 3);
var _FPDFAnnot_RemoveInkList = Module['_FPDFAnnot_RemoveInkList'] = createExportWrapper('FPDFAnnot_RemoveInkList', 1);
var _FPDFAnnot_AppendObject = Module['_FPDFAnnot_AppendObject'] = createExportWrapper('FPDFAnnot_AppendObject', 2);
var _FPDFAnnot_GetObjectCount = Module['_FPDFAnnot_GetObjectCount'] = createExportWrapper('FPDFAnnot_GetObjectCount', 1);
var _FPDFAnnot_GetObject = Module['_FPDFAnnot_GetObject'] = createExportWrapper('FPDFAnnot_GetObject', 2);
var _FPDFAnnot_RemoveObject = Module['_FPDFAnnot_RemoveObject'] = createExportWrapper('FPDFAnnot_RemoveObject', 2);
var _FPDFAnnot_SetColor = Module['_FPDFAnnot_SetColor'] = createExportWrapper('FPDFAnnot_SetColor', 6);
var _FPDFAnnot_GetColor = Module['_FPDFAnnot_GetColor'] = createExportWrapper('FPDFAnnot_GetColor', 6);
var _FPDFAnnot_HasAttachmentPoints = Module['_FPDFAnnot_HasAttachmentPoints'] = createExportWrapper('FPDFAnnot_HasAttachmentPoints', 1);
var _FPDFAnnot_SetAttachmentPoints = Module['_FPDFAnnot_SetAttachmentPoints'] = createExportWrapper('FPDFAnnot_SetAttachmentPoints', 3);
var _FPDFAnnot_AppendAttachmentPoints = Module['_FPDFAnnot_AppendAttachmentPoints'] = createExportWrapper('FPDFAnnot_AppendAttachmentPoints', 2);
var _FPDFAnnot_CountAttachmentPoints = Module['_FPDFAnnot_CountAttachmentPoints'] = createExportWrapper('FPDFAnnot_CountAttachmentPoints', 1);
var _FPDFAnnot_GetAttachmentPoints = Module['_FPDFAnnot_GetAttachmentPoints'] = createExportWrapper('FPDFAnnot_GetAttachmentPoints', 3);
var _FPDFAnnot_SetRect = Module['_FPDFAnnot_SetRect'] = createExportWrapper('FPDFAnnot_SetRect', 2);
var _FPDFAnnot_GetRect = Module['_FPDFAnnot_GetRect'] = createExportWrapper('FPDFAnnot_GetRect', 2);
var _FPDFAnnot_GetVertices = Module['_FPDFAnnot_GetVertices'] = createExportWrapper('FPDFAnnot_GetVertices', 3);
var _FPDFAnnot_GetInkListCount = Module['_FPDFAnnot_GetInkListCount'] = createExportWrapper('FPDFAnnot_GetInkListCount', 1);
var _FPDFAnnot_GetInkListPath = Module['_FPDFAnnot_GetInkListPath'] = createExportWrapper('FPDFAnnot_GetInkListPath', 4);
var _FPDFAnnot_GetLine = Module['_FPDFAnnot_GetLine'] = createExportWrapper('FPDFAnnot_GetLine', 3);
var _FPDFAnnot_SetBorder = Module['_FPDFAnnot_SetBorder'] = createExportWrapper('FPDFAnnot_SetBorder', 4);
var _FPDFAnnot_GetBorder = Module['_FPDFAnnot_GetBorder'] = createExportWrapper('FPDFAnnot_GetBorder', 4);
var _FPDFAnnot_HasKey = Module['_FPDFAnnot_HasKey'] = createExportWrapper('FPDFAnnot_HasKey', 2);
var _FPDFAnnot_GetValueType = Module['_FPDFAnnot_GetValueType'] = createExportWrapper('FPDFAnnot_GetValueType', 2);
var _FPDFAnnot_SetStringValue = Module['_FPDFAnnot_SetStringValue'] = createExportWrapper('FPDFAnnot_SetStringValue', 3);
var _FPDFAnnot_GetStringValue = Module['_FPDFAnnot_GetStringValue'] = createExportWrapper('FPDFAnnot_GetStringValue', 4);
var _FPDFAnnot_GetNumberValue = Module['_FPDFAnnot_GetNumberValue'] = createExportWrapper('FPDFAnnot_GetNumberValue', 3);
var _FPDFAnnot_SetAP = Module['_FPDFAnnot_SetAP'] = createExportWrapper('FPDFAnnot_SetAP', 3);
var _FPDFAnnot_GetAP = Module['_FPDFAnnot_GetAP'] = createExportWrapper('FPDFAnnot_GetAP', 4);
var _FPDFAnnot_GetLinkedAnnot = Module['_FPDFAnnot_GetLinkedAnnot'] = createExportWrapper('FPDFAnnot_GetLinkedAnnot', 2);
var _FPDFAnnot_GetFlags = Module['_FPDFAnnot_GetFlags'] = createExportWrapper('FPDFAnnot_GetFlags', 1);
var _FPDFAnnot_SetFlags = Module['_FPDFAnnot_SetFlags'] = createExportWrapper('FPDFAnnot_SetFlags', 2);
var _FPDFAnnot_GetFormFieldFlags = Module['_FPDFAnnot_GetFormFieldFlags'] = createExportWrapper('FPDFAnnot_GetFormFieldFlags', 2);
var _FPDFAnnot_SetFormFieldFlags = Module['_FPDFAnnot_SetFormFieldFlags'] = createExportWrapper('FPDFAnnot_SetFormFieldFlags', 3);
var _FPDFAnnot_GetFormFieldAtPoint = Module['_FPDFAnnot_GetFormFieldAtPoint'] = createExportWrapper('FPDFAnnot_GetFormFieldAtPoint', 3);
var _FPDFAnnot_GetFormFieldName = Module['_FPDFAnnot_GetFormFieldName'] = createExportWrapper('FPDFAnnot_GetFormFieldName', 4);
var _FPDFAnnot_GetFormFieldType = Module['_FPDFAnnot_GetFormFieldType'] = createExportWrapper('FPDFAnnot_GetFormFieldType', 2);
var _FPDFAnnot_GetFormAdditionalActionJavaScript = Module['_FPDFAnnot_GetFormAdditionalActionJavaScript'] = createExportWrapper('FPDFAnnot_GetFormAdditionalActionJavaScript', 5);
var _FPDFAnnot_GetFormFieldAlternateName = Module['_FPDFAnnot_GetFormFieldAlternateName'] = createExportWrapper('FPDFAnnot_GetFormFieldAlternateName', 4);
var _FPDFAnnot_GetFormFieldValue = Module['_FPDFAnnot_GetFormFieldValue'] = createExportWrapper('FPDFAnnot_GetFormFieldValue', 4);
var _FPDFAnnot_GetOptionCount = Module['_FPDFAnnot_GetOptionCount'] = createExportWrapper('FPDFAnnot_GetOptionCount', 2);
var _FPDFAnnot_GetOptionLabel = Module['_FPDFAnnot_GetOptionLabel'] = createExportWrapper('FPDFAnnot_GetOptionLabel', 5);
var _FPDFAnnot_IsOptionSelected = Module['_FPDFAnnot_IsOptionSelected'] = createExportWrapper('FPDFAnnot_IsOptionSelected', 3);
var _FPDFAnnot_GetFontSize = Module['_FPDFAnnot_GetFontSize'] = createExportWrapper('FPDFAnnot_GetFontSize', 3);
var _FPDFAnnot_SetFontColor = Module['_FPDFAnnot_SetFontColor'] = createExportWrapper('FPDFAnnot_SetFontColor', 5);
var _FPDFAnnot_GetFontColor = Module['_FPDFAnnot_GetFontColor'] = createExportWrapper('FPDFAnnot_GetFontColor', 5);
var _FPDFAnnot_IsChecked = Module['_FPDFAnnot_IsChecked'] = createExportWrapper('FPDFAnnot_IsChecked', 2);
var _FPDFAnnot_SetFocusableSubtypes = Module['_FPDFAnnot_SetFocusableSubtypes'] = createExportWrapper('FPDFAnnot_SetFocusableSubtypes', 3);
var _FPDFAnnot_GetFocusableSubtypesCount = Module['_FPDFAnnot_GetFocusableSubtypesCount'] = createExportWrapper('FPDFAnnot_GetFocusableSubtypesCount', 1);
var _FPDFAnnot_GetFocusableSubtypes = Module['_FPDFAnnot_GetFocusableSubtypes'] = createExportWrapper('FPDFAnnot_GetFocusableSubtypes', 3);
var _FPDFAnnot_GetLink = Module['_FPDFAnnot_GetLink'] = createExportWrapper('FPDFAnnot_GetLink', 1);
var _FPDFAnnot_GetFormControlCount = Module['_FPDFAnnot_GetFormControlCount'] = createExportWrapper('FPDFAnnot_GetFormControlCount', 2);
var _FPDFAnnot_GetFormControlIndex = Module['_FPDFAnnot_GetFormControlIndex'] = createExportWrapper('FPDFAnnot_GetFormControlIndex', 2);
var _FPDFAnnot_GetFormFieldExportValue = Module['_FPDFAnnot_GetFormFieldExportValue'] = createExportWrapper('FPDFAnnot_GetFormFieldExportValue', 4);
var _FPDFAnnot_SetURI = Module['_FPDFAnnot_SetURI'] = createExportWrapper('FPDFAnnot_SetURI', 2);
var _FPDFAnnot_GetFileAttachment = Module['_FPDFAnnot_GetFileAttachment'] = createExportWrapper('FPDFAnnot_GetFileAttachment', 1);
var _FPDFAnnot_AddFileAttachment = Module['_FPDFAnnot_AddFileAttachment'] = createExportWrapper('FPDFAnnot_AddFileAttachment', 2);
var _EPDFAnnot_SetColor = Module['_EPDFAnnot_SetColor'] = createExportWrapper('EPDFAnnot_SetColor', 5);
var _EPDFAnnot_GetColor = Module['_EPDFAnnot_GetColor'] = createExportWrapper('EPDFAnnot_GetColor', 5);
var _EPDFAnnot_ClearColor = Module['_EPDFAnnot_ClearColor'] = createExportWrapper('EPDFAnnot_ClearColor', 2);
var _EPDFAnnot_SetOpacity = Module['_EPDFAnnot_SetOpacity'] = createExportWrapper('EPDFAnnot_SetOpacity', 2);
var _EPDFAnnot_GetOpacity = Module['_EPDFAnnot_GetOpacity'] = createExportWrapper('EPDFAnnot_GetOpacity', 2);
var _EPDFAnnot_GetBorderEffect = Module['_EPDFAnnot_GetBorderEffect'] = createExportWrapper('EPDFAnnot_GetBorderEffect', 2);
var _EPDFAnnot_GetRectangleDifferences = Module['_EPDFAnnot_GetRectangleDifferences'] = createExportWrapper('EPDFAnnot_GetRectangleDifferences', 5);
var _EPDFAnnot_GetBorderDashPatternCount = Module['_EPDFAnnot_GetBorderDashPatternCount'] = createExportWrapper('EPDFAnnot_GetBorderDashPatternCount', 1);
var _EPDFAnnot_GetBorderDashPattern = Module['_EPDFAnnot_GetBorderDashPattern'] = createExportWrapper('EPDFAnnot_GetBorderDashPattern', 3);
var _EPDFAnnot_SetBorderDashPattern = Module['_EPDFAnnot_SetBorderDashPattern'] = createExportWrapper('EPDFAnnot_SetBorderDashPattern', 3);
var _EPDFAnnot_GetBorderStyle = Module['_EPDFAnnot_GetBorderStyle'] = createExportWrapper('EPDFAnnot_GetBorderStyle', 2);
var _EPDFAnnot_SetBorderStyle = Module['_EPDFAnnot_SetBorderStyle'] = createExportWrapper('EPDFAnnot_SetBorderStyle', 3);
var _EPDFAnnot_GenerateAppearance = Module['_EPDFAnnot_GenerateAppearance'] = createExportWrapper('EPDFAnnot_GenerateAppearance', 1);
var _EPDFAnnot_GenerateAppearanceWithBlend = Module['_EPDFAnnot_GenerateAppearanceWithBlend'] = createExportWrapper('EPDFAnnot_GenerateAppearanceWithBlend', 2);
var _EPDFAnnot_GetBlendMode = Module['_EPDFAnnot_GetBlendMode'] = createExportWrapper('EPDFAnnot_GetBlendMode', 1);
var _EPDFAnnot_SetIntent = Module['_EPDFAnnot_SetIntent'] = createExportWrapper('EPDFAnnot_SetIntent', 2);
var _EPDFAnnot_GetIntent = Module['_EPDFAnnot_GetIntent'] = createExportWrapper('EPDFAnnot_GetIntent', 3);
var _EPDFAnnot_GetRichContent = Module['_EPDFAnnot_GetRichContent'] = createExportWrapper('EPDFAnnot_GetRichContent', 3);
var _EPDFAnnot_SetLineEndings = Module['_EPDFAnnot_SetLineEndings'] = createExportWrapper('EPDFAnnot_SetLineEndings', 3);
var _EPDFAnnot_GetLineEndings = Module['_EPDFAnnot_GetLineEndings'] = createExportWrapper('EPDFAnnot_GetLineEndings', 3);
var _EPDFAnnot_SetVertices = Module['_EPDFAnnot_SetVertices'] = createExportWrapper('EPDFAnnot_SetVertices', 3);
var _EPDFAnnot_SetLine = Module['_EPDFAnnot_SetLine'] = createExportWrapper('EPDFAnnot_SetLine', 3);
var _EPDFAnnot_SetDefaultAppearance = Module['_EPDFAnnot_SetDefaultAppearance'] = createExportWrapper('EPDFAnnot_SetDefaultAppearance', 6);
var _EPDFAnnot_GetDefaultAppearance = Module['_EPDFAnnot_GetDefaultAppearance'] = createExportWrapper('EPDFAnnot_GetDefaultAppearance', 6);
var _EPDFAnnot_SetTextAlignment = Module['_EPDFAnnot_SetTextAlignment'] = createExportWrapper('EPDFAnnot_SetTextAlignment', 2);
var _EPDFAnnot_GetTextAlignment = Module['_EPDFAnnot_GetTextAlignment'] = createExportWrapper('EPDFAnnot_GetTextAlignment', 1);
var _EPDFAnnot_SetVerticalAlignment = Module['_EPDFAnnot_SetVerticalAlignment'] = createExportWrapper('EPDFAnnot_SetVerticalAlignment', 2);
var _EPDFAnnot_GetVerticalAlignment = Module['_EPDFAnnot_GetVerticalAlignment'] = createExportWrapper('EPDFAnnot_GetVerticalAlignment', 1);
var _EPDFPage_GetAnnotByName = Module['_EPDFPage_GetAnnotByName'] = createExportWrapper('EPDFPage_GetAnnotByName', 2);
var _EPDFPage_RemoveAnnotByName = Module['_EPDFPage_RemoveAnnotByName'] = createExportWrapper('EPDFPage_RemoveAnnotByName', 2);
var _EPDFAnnot_SetLinkedAnnot = Module['_EPDFAnnot_SetLinkedAnnot'] = createExportWrapper('EPDFAnnot_SetLinkedAnnot', 3);
var _EPDFPage_GetAnnotCountRaw = Module['_EPDFPage_GetAnnotCountRaw'] = createExportWrapper('EPDFPage_GetAnnotCountRaw', 2);
var _EPDFPage_GetAnnotRaw = Module['_EPDFPage_GetAnnotRaw'] = createExportWrapper('EPDFPage_GetAnnotRaw', 3);
var _EPDFPage_RemoveAnnotRaw = Module['_EPDFPage_RemoveAnnotRaw'] = createExportWrapper('EPDFPage_RemoveAnnotRaw', 3);
var _EPDFAnnot_SetIcon = Module['_EPDFAnnot_SetIcon'] = createExportWrapper('EPDFAnnot_SetIcon', 2);
var _EPDFAnnot_GetIcon = Module['_EPDFAnnot_GetIcon'] = createExportWrapper('EPDFAnnot_GetIcon', 1);
var _EPDFAnnot_UpdateAppearanceToRect = Module['_EPDFAnnot_UpdateAppearanceToRect'] = createExportWrapper('EPDFAnnot_UpdateAppearanceToRect', 2);
var _EPDFPage_CreateAnnot = Module['_EPDFPage_CreateAnnot'] = createExportWrapper('EPDFPage_CreateAnnot', 2);
var _FPDFDoc_GetAttachmentCount = Module['_FPDFDoc_GetAttachmentCount'] = createExportWrapper('FPDFDoc_GetAttachmentCount', 1);
var _FPDFDoc_AddAttachment = Module['_FPDFDoc_AddAttachment'] = createExportWrapper('FPDFDoc_AddAttachment', 2);
var _FPDFDoc_GetAttachment = Module['_FPDFDoc_GetAttachment'] = createExportWrapper('FPDFDoc_GetAttachment', 2);
var _FPDFDoc_DeleteAttachment = Module['_FPDFDoc_DeleteAttachment'] = createExportWrapper('FPDFDoc_DeleteAttachment', 2);
var _FPDFAttachment_GetName = Module['_FPDFAttachment_GetName'] = createExportWrapper('FPDFAttachment_GetName', 3);
var _FPDFAttachment_HasKey = Module['_FPDFAttachment_HasKey'] = createExportWrapper('FPDFAttachment_HasKey', 2);
var _FPDFAttachment_GetValueType = Module['_FPDFAttachment_GetValueType'] = createExportWrapper('FPDFAttachment_GetValueType', 2);
var _FPDFAttachment_SetStringValue = Module['_FPDFAttachment_SetStringValue'] = createExportWrapper('FPDFAttachment_SetStringValue', 3);
var _FPDFAttachment_GetStringValue = Module['_FPDFAttachment_GetStringValue'] = createExportWrapper('FPDFAttachment_GetStringValue', 4);
var _FPDFAttachment_SetFile = Module['_FPDFAttachment_SetFile'] = createExportWrapper('FPDFAttachment_SetFile', 4);
var _FPDFAttachment_GetFile = Module['_FPDFAttachment_GetFile'] = createExportWrapper('FPDFAttachment_GetFile', 4);
var _FPDFAttachment_GetSubtype = Module['_FPDFAttachment_GetSubtype'] = createExportWrapper('FPDFAttachment_GetSubtype', 3);
var _EPDFAttachment_SetSubtype = Module['_EPDFAttachment_SetSubtype'] = createExportWrapper('EPDFAttachment_SetSubtype', 2);
var _EPDFAttachment_SetDescription = Module['_EPDFAttachment_SetDescription'] = createExportWrapper('EPDFAttachment_SetDescription', 2);
var _EPDFAttachment_GetDescription = Module['_EPDFAttachment_GetDescription'] = createExportWrapper('EPDFAttachment_GetDescription', 3);
var _EPDFAttachment_GetIntegerValue = Module['_EPDFAttachment_GetIntegerValue'] = createExportWrapper('EPDFAttachment_GetIntegerValue', 3);
var _FPDFCatalog_IsTagged = Module['_FPDFCatalog_IsTagged'] = createExportWrapper('FPDFCatalog_IsTagged', 1);
var _FPDFCatalog_SetLanguage = Module['_FPDFCatalog_SetLanguage'] = createExportWrapper('FPDFCatalog_SetLanguage', 2);
var _EPDFCatalog_GetLanguage = Module['_EPDFCatalog_GetLanguage'] = createExportWrapper('EPDFCatalog_GetLanguage', 3);
var _FPDFAvail_Create = Module['_FPDFAvail_Create'] = createExportWrapper('FPDFAvail_Create', 2);
var _FPDFAvail_Destroy = Module['_FPDFAvail_Destroy'] = createExportWrapper('FPDFAvail_Destroy', 1);
var _FPDFAvail_IsDocAvail = Module['_FPDFAvail_IsDocAvail'] = createExportWrapper('FPDFAvail_IsDocAvail', 2);
var _FPDFAvail_GetDocument = Module['_FPDFAvail_GetDocument'] = createExportWrapper('FPDFAvail_GetDocument', 2);
var _FPDFAvail_GetFirstPageNum = Module['_FPDFAvail_GetFirstPageNum'] = createExportWrapper('FPDFAvail_GetFirstPageNum', 1);
var _FPDFAvail_IsPageAvail = Module['_FPDFAvail_IsPageAvail'] = createExportWrapper('FPDFAvail_IsPageAvail', 3);
var _FPDFAvail_IsFormAvail = Module['_FPDFAvail_IsFormAvail'] = createExportWrapper('FPDFAvail_IsFormAvail', 2);
var _FPDFAvail_IsLinearized = Module['_FPDFAvail_IsLinearized'] = createExportWrapper('FPDFAvail_IsLinearized', 1);
var _FPDFBookmark_GetFirstChild = Module['_FPDFBookmark_GetFirstChild'] = createExportWrapper('FPDFBookmark_GetFirstChild', 2);
var _FPDFBookmark_GetNextSibling = Module['_FPDFBookmark_GetNextSibling'] = createExportWrapper('FPDFBookmark_GetNextSibling', 2);
var _FPDFBookmark_GetTitle = Module['_FPDFBookmark_GetTitle'] = createExportWrapper('FPDFBookmark_GetTitle', 3);
var _FPDFBookmark_GetCount = Module['_FPDFBookmark_GetCount'] = createExportWrapper('FPDFBookmark_GetCount', 1);
var _FPDFBookmark_Find = Module['_FPDFBookmark_Find'] = createExportWrapper('FPDFBookmark_Find', 2);
var _FPDFBookmark_GetDest = Module['_FPDFBookmark_GetDest'] = createExportWrapper('FPDFBookmark_GetDest', 2);
var _FPDFBookmark_GetAction = Module['_FPDFBookmark_GetAction'] = createExportWrapper('FPDFBookmark_GetAction', 1);
var _FPDFAction_GetType = Module['_FPDFAction_GetType'] = createExportWrapper('FPDFAction_GetType', 1);
var _FPDFAction_GetDest = Module['_FPDFAction_GetDest'] = createExportWrapper('FPDFAction_GetDest', 2);
var _FPDFAction_GetFilePath = Module['_FPDFAction_GetFilePath'] = createExportWrapper('FPDFAction_GetFilePath', 3);
var _FPDFAction_GetURIPath = Module['_FPDFAction_GetURIPath'] = createExportWrapper('FPDFAction_GetURIPath', 4);
var _FPDFDest_GetDestPageIndex = Module['_FPDFDest_GetDestPageIndex'] = createExportWrapper('FPDFDest_GetDestPageIndex', 2);
var _FPDFDest_GetView = Module['_FPDFDest_GetView'] = createExportWrapper('FPDFDest_GetView', 3);
var _FPDFDest_GetLocationInPage = Module['_FPDFDest_GetLocationInPage'] = createExportWrapper('FPDFDest_GetLocationInPage', 7);
var _FPDFLink_GetLinkAtPoint = Module['_FPDFLink_GetLinkAtPoint'] = createExportWrapper('FPDFLink_GetLinkAtPoint', 3);
var _FPDFLink_GetLinkZOrderAtPoint = Module['_FPDFLink_GetLinkZOrderAtPoint'] = createExportWrapper('FPDFLink_GetLinkZOrderAtPoint', 3);
var _FPDFLink_GetDest = Module['_FPDFLink_GetDest'] = createExportWrapper('FPDFLink_GetDest', 2);
var _FPDFLink_GetAction = Module['_FPDFLink_GetAction'] = createExportWrapper('FPDFLink_GetAction', 1);
var _FPDFLink_Enumerate = Module['_FPDFLink_Enumerate'] = createExportWrapper('FPDFLink_Enumerate', 3);
var _FPDFLink_GetAnnot = Module['_FPDFLink_GetAnnot'] = createExportWrapper('FPDFLink_GetAnnot', 2);
var _FPDFLink_GetAnnotRect = Module['_FPDFLink_GetAnnotRect'] = createExportWrapper('FPDFLink_GetAnnotRect', 2);
var _FPDFLink_CountQuadPoints = Module['_FPDFLink_CountQuadPoints'] = createExportWrapper('FPDFLink_CountQuadPoints', 1);
var _FPDFLink_GetQuadPoints = Module['_FPDFLink_GetQuadPoints'] = createExportWrapper('FPDFLink_GetQuadPoints', 3);
var _FPDF_GetPageAAction = Module['_FPDF_GetPageAAction'] = createExportWrapper('FPDF_GetPageAAction', 2);
var _FPDF_GetFileIdentifier = Module['_FPDF_GetFileIdentifier'] = createExportWrapper('FPDF_GetFileIdentifier', 4);
var _FPDF_GetMetaText = Module['_FPDF_GetMetaText'] = createExportWrapper('FPDF_GetMetaText', 4);
var _FPDF_GetPageLabel = Module['_FPDF_GetPageLabel'] = createExportWrapper('FPDF_GetPageLabel', 4);
var _EPDF_SetMetaText = Module['_EPDF_SetMetaText'] = createExportWrapper('EPDF_SetMetaText', 3);
var _EPDF_HasMetaText = Module['_EPDF_HasMetaText'] = createExportWrapper('EPDF_HasMetaText', 2);
var _EPDF_GetMetaTrapped = Module['_EPDF_GetMetaTrapped'] = createExportWrapper('EPDF_GetMetaTrapped', 1);
var _EPDF_SetMetaTrapped = Module['_EPDF_SetMetaTrapped'] = createExportWrapper('EPDF_SetMetaTrapped', 2);
var _EPDF_GetMetaKeyCount = Module['_EPDF_GetMetaKeyCount'] = createExportWrapper('EPDF_GetMetaKeyCount', 2);
var _EPDF_GetMetaKeyName = Module['_EPDF_GetMetaKeyName'] = createExportWrapper('EPDF_GetMetaKeyName', 5);
var _FPDFPageObj_NewImageObj = Module['_FPDFPageObj_NewImageObj'] = createExportWrapper('FPDFPageObj_NewImageObj', 1);
var _FPDFImageObj_LoadJpegFile = Module['_FPDFImageObj_LoadJpegFile'] = createExportWrapper('FPDFImageObj_LoadJpegFile', 4);
var _FPDFImageObj_LoadJpegFileInline = Module['_FPDFImageObj_LoadJpegFileInline'] = createExportWrapper('FPDFImageObj_LoadJpegFileInline', 4);
var _FPDFImageObj_SetMatrix = Module['_FPDFImageObj_SetMatrix'] = createExportWrapper('FPDFImageObj_SetMatrix', 7);
var _FPDFImageObj_SetBitmap = Module['_FPDFImageObj_SetBitmap'] = createExportWrapper('FPDFImageObj_SetBitmap', 4);
var _FPDFImageObj_GetBitmap = Module['_FPDFImageObj_GetBitmap'] = createExportWrapper('FPDFImageObj_GetBitmap', 1);
var _FPDFImageObj_GetRenderedBitmap = Module['_FPDFImageObj_GetRenderedBitmap'] = createExportWrapper('FPDFImageObj_GetRenderedBitmap', 3);
var _FPDFImageObj_GetImageDataDecoded = Module['_FPDFImageObj_GetImageDataDecoded'] = createExportWrapper('FPDFImageObj_GetImageDataDecoded', 3);
var _FPDFImageObj_GetImageDataRaw = Module['_FPDFImageObj_GetImageDataRaw'] = createExportWrapper('FPDFImageObj_GetImageDataRaw', 3);
var _FPDFImageObj_GetImageFilterCount = Module['_FPDFImageObj_GetImageFilterCount'] = createExportWrapper('FPDFImageObj_GetImageFilterCount', 1);
var _FPDFImageObj_GetImageFilter = Module['_FPDFImageObj_GetImageFilter'] = createExportWrapper('FPDFImageObj_GetImageFilter', 4);
var _FPDFImageObj_GetImageMetadata = Module['_FPDFImageObj_GetImageMetadata'] = createExportWrapper('FPDFImageObj_GetImageMetadata', 3);
var _FPDFImageObj_GetImagePixelSize = Module['_FPDFImageObj_GetImagePixelSize'] = createExportWrapper('FPDFImageObj_GetImagePixelSize', 3);
var _FPDFImageObj_GetIccProfileDataDecoded = Module['_FPDFImageObj_GetIccProfileDataDecoded'] = createExportWrapper('FPDFImageObj_GetIccProfileDataDecoded', 5);
var _FPDF_CreateNewDocument = Module['_FPDF_CreateNewDocument'] = createExportWrapper('FPDF_CreateNewDocument', 0);
var _FPDFPage_Delete = Module['_FPDFPage_Delete'] = createExportWrapper('FPDFPage_Delete', 2);
var _FPDF_MovePages = Module['_FPDF_MovePages'] = createExportWrapper('FPDF_MovePages', 4);
var _FPDFPage_New = Module['_FPDFPage_New'] = createExportWrapper('FPDFPage_New', 4);
var _FPDFPage_GetRotation = Module['_FPDFPage_GetRotation'] = createExportWrapper('FPDFPage_GetRotation', 1);
var _FPDFPage_InsertObject = Module['_FPDFPage_InsertObject'] = createExportWrapper('FPDFPage_InsertObject', 2);
var _FPDFPage_InsertObjectAtIndex = Module['_FPDFPage_InsertObjectAtIndex'] = createExportWrapper('FPDFPage_InsertObjectAtIndex', 3);
var _FPDFPage_RemoveObject = Module['_FPDFPage_RemoveObject'] = createExportWrapper('FPDFPage_RemoveObject', 2);
var _FPDFPage_CountObjects = Module['_FPDFPage_CountObjects'] = createExportWrapper('FPDFPage_CountObjects', 1);
var _FPDFPage_GetObject = Module['_FPDFPage_GetObject'] = createExportWrapper('FPDFPage_GetObject', 2);
var _FPDFPage_HasTransparency = Module['_FPDFPage_HasTransparency'] = createExportWrapper('FPDFPage_HasTransparency', 1);
var _FPDFPageObj_Destroy = Module['_FPDFPageObj_Destroy'] = createExportWrapper('FPDFPageObj_Destroy', 1);
var _FPDFPageObj_GetMarkedContentID = Module['_FPDFPageObj_GetMarkedContentID'] = createExportWrapper('FPDFPageObj_GetMarkedContentID', 1);
var _FPDFPageObj_CountMarks = Module['_FPDFPageObj_CountMarks'] = createExportWrapper('FPDFPageObj_CountMarks', 1);
var _FPDFPageObj_GetMark = Module['_FPDFPageObj_GetMark'] = createExportWrapper('FPDFPageObj_GetMark', 2);
var _FPDFPageObj_AddMark = Module['_FPDFPageObj_AddMark'] = createExportWrapper('FPDFPageObj_AddMark', 2);
var _FPDFPageObj_RemoveMark = Module['_FPDFPageObj_RemoveMark'] = createExportWrapper('FPDFPageObj_RemoveMark', 2);
var _FPDFPageObjMark_GetName = Module['_FPDFPageObjMark_GetName'] = createExportWrapper('FPDFPageObjMark_GetName', 4);
var _FPDFPageObjMark_CountParams = Module['_FPDFPageObjMark_CountParams'] = createExportWrapper('FPDFPageObjMark_CountParams', 1);
var _FPDFPageObjMark_GetParamKey = Module['_FPDFPageObjMark_GetParamKey'] = createExportWrapper('FPDFPageObjMark_GetParamKey', 5);
var _FPDFPageObjMark_GetParamValueType = Module['_FPDFPageObjMark_GetParamValueType'] = createExportWrapper('FPDFPageObjMark_GetParamValueType', 2);
var _FPDFPageObjMark_GetParamIntValue = Module['_FPDFPageObjMark_GetParamIntValue'] = createExportWrapper('FPDFPageObjMark_GetParamIntValue', 3);
var _FPDFPageObjMark_GetParamStringValue = Module['_FPDFPageObjMark_GetParamStringValue'] = createExportWrapper('FPDFPageObjMark_GetParamStringValue', 5);
var _FPDFPageObjMark_GetParamBlobValue = Module['_FPDFPageObjMark_GetParamBlobValue'] = createExportWrapper('FPDFPageObjMark_GetParamBlobValue', 5);
var _FPDFPageObj_HasTransparency = Module['_FPDFPageObj_HasTransparency'] = createExportWrapper('FPDFPageObj_HasTransparency', 1);
var _FPDFPageObjMark_SetIntParam = Module['_FPDFPageObjMark_SetIntParam'] = createExportWrapper('FPDFPageObjMark_SetIntParam', 5);
var _FPDFPageObjMark_SetStringParam = Module['_FPDFPageObjMark_SetStringParam'] = createExportWrapper('FPDFPageObjMark_SetStringParam', 5);
var _FPDFPageObjMark_SetBlobParam = Module['_FPDFPageObjMark_SetBlobParam'] = createExportWrapper('FPDFPageObjMark_SetBlobParam', 6);
var _FPDFPageObjMark_RemoveParam = Module['_FPDFPageObjMark_RemoveParam'] = createExportWrapper('FPDFPageObjMark_RemoveParam', 3);
var _FPDFPageObj_GetType = Module['_FPDFPageObj_GetType'] = createExportWrapper('FPDFPageObj_GetType', 1);
var _FPDFPageObj_GetIsActive = Module['_FPDFPageObj_GetIsActive'] = createExportWrapper('FPDFPageObj_GetIsActive', 2);
var _FPDFPageObj_SetIsActive = Module['_FPDFPageObj_SetIsActive'] = createExportWrapper('FPDFPageObj_SetIsActive', 2);
var _FPDFPage_GenerateContent = Module['_FPDFPage_GenerateContent'] = createExportWrapper('FPDFPage_GenerateContent', 1);
var _FPDFPageObj_Transform = Module['_FPDFPageObj_Transform'] = createExportWrapper('FPDFPageObj_Transform', 7);
var _FPDFPageObj_TransformF = Module['_FPDFPageObj_TransformF'] = createExportWrapper('FPDFPageObj_TransformF', 2);
var _FPDFPageObj_GetMatrix = Module['_FPDFPageObj_GetMatrix'] = createExportWrapper('FPDFPageObj_GetMatrix', 2);
var _FPDFPageObj_SetMatrix = Module['_FPDFPageObj_SetMatrix'] = createExportWrapper('FPDFPageObj_SetMatrix', 2);
var _FPDFPageObj_SetBlendMode = Module['_FPDFPageObj_SetBlendMode'] = createExportWrapper('FPDFPageObj_SetBlendMode', 2);
var _FPDFPage_TransformAnnots = Module['_FPDFPage_TransformAnnots'] = createExportWrapper('FPDFPage_TransformAnnots', 7);
var _FPDFPage_SetRotation = Module['_FPDFPage_SetRotation'] = createExportWrapper('FPDFPage_SetRotation', 2);
var _FPDFPageObj_SetFillColor = Module['_FPDFPageObj_SetFillColor'] = createExportWrapper('FPDFPageObj_SetFillColor', 5);
var _FPDFPageObj_GetFillColor = Module['_FPDFPageObj_GetFillColor'] = createExportWrapper('FPDFPageObj_GetFillColor', 5);
var _FPDFPageObj_GetBounds = Module['_FPDFPageObj_GetBounds'] = createExportWrapper('FPDFPageObj_GetBounds', 5);
var _FPDFPageObj_GetRotatedBounds = Module['_FPDFPageObj_GetRotatedBounds'] = createExportWrapper('FPDFPageObj_GetRotatedBounds', 2);
var _FPDFPageObj_SetStrokeColor = Module['_FPDFPageObj_SetStrokeColor'] = createExportWrapper('FPDFPageObj_SetStrokeColor', 5);
var _FPDFPageObj_GetStrokeColor = Module['_FPDFPageObj_GetStrokeColor'] = createExportWrapper('FPDFPageObj_GetStrokeColor', 5);
var _FPDFPageObj_SetStrokeWidth = Module['_FPDFPageObj_SetStrokeWidth'] = createExportWrapper('FPDFPageObj_SetStrokeWidth', 2);
var _FPDFPageObj_GetStrokeWidth = Module['_FPDFPageObj_GetStrokeWidth'] = createExportWrapper('FPDFPageObj_GetStrokeWidth', 2);
var _FPDFPageObj_GetLineJoin = Module['_FPDFPageObj_GetLineJoin'] = createExportWrapper('FPDFPageObj_GetLineJoin', 1);
var _FPDFPageObj_SetLineJoin = Module['_FPDFPageObj_SetLineJoin'] = createExportWrapper('FPDFPageObj_SetLineJoin', 2);
var _FPDFPageObj_GetLineCap = Module['_FPDFPageObj_GetLineCap'] = createExportWrapper('FPDFPageObj_GetLineCap', 1);
var _FPDFPageObj_SetLineCap = Module['_FPDFPageObj_SetLineCap'] = createExportWrapper('FPDFPageObj_SetLineCap', 2);
var _FPDFPageObj_GetDashPhase = Module['_FPDFPageObj_GetDashPhase'] = createExportWrapper('FPDFPageObj_GetDashPhase', 2);
var _FPDFPageObj_SetDashPhase = Module['_FPDFPageObj_SetDashPhase'] = createExportWrapper('FPDFPageObj_SetDashPhase', 2);
var _FPDFPageObj_GetDashCount = Module['_FPDFPageObj_GetDashCount'] = createExportWrapper('FPDFPageObj_GetDashCount', 1);
var _FPDFPageObj_GetDashArray = Module['_FPDFPageObj_GetDashArray'] = createExportWrapper('FPDFPageObj_GetDashArray', 3);
var _FPDFPageObj_SetDashArray = Module['_FPDFPageObj_SetDashArray'] = createExportWrapper('FPDFPageObj_SetDashArray', 4);
var _FPDFFormObj_CountObjects = Module['_FPDFFormObj_CountObjects'] = createExportWrapper('FPDFFormObj_CountObjects', 1);
var _FPDFFormObj_GetObject = Module['_FPDFFormObj_GetObject'] = createExportWrapper('FPDFFormObj_GetObject', 2);
var _FPDFFormObj_RemoveObject = Module['_FPDFFormObj_RemoveObject'] = createExportWrapper('FPDFFormObj_RemoveObject', 2);
var _FPDFPageObj_CreateNewPath = Module['_FPDFPageObj_CreateNewPath'] = createExportWrapper('FPDFPageObj_CreateNewPath', 2);
var _FPDFPageObj_CreateNewRect = Module['_FPDFPageObj_CreateNewRect'] = createExportWrapper('FPDFPageObj_CreateNewRect', 4);
var _FPDFPath_CountSegments = Module['_FPDFPath_CountSegments'] = createExportWrapper('FPDFPath_CountSegments', 1);
var _FPDFPath_GetPathSegment = Module['_FPDFPath_GetPathSegment'] = createExportWrapper('FPDFPath_GetPathSegment', 2);
var _FPDFPath_MoveTo = Module['_FPDFPath_MoveTo'] = createExportWrapper('FPDFPath_MoveTo', 3);
var _FPDFPath_LineTo = Module['_FPDFPath_LineTo'] = createExportWrapper('FPDFPath_LineTo', 3);
var _FPDFPath_BezierTo = Module['_FPDFPath_BezierTo'] = createExportWrapper('FPDFPath_BezierTo', 7);
var _FPDFPath_Close = Module['_FPDFPath_Close'] = createExportWrapper('FPDFPath_Close', 1);
var _FPDFPath_SetDrawMode = Module['_FPDFPath_SetDrawMode'] = createExportWrapper('FPDFPath_SetDrawMode', 3);
var _FPDFPath_GetDrawMode = Module['_FPDFPath_GetDrawMode'] = createExportWrapper('FPDFPath_GetDrawMode', 3);
var _FPDFPathSegment_GetPoint = Module['_FPDFPathSegment_GetPoint'] = createExportWrapper('FPDFPathSegment_GetPoint', 3);
var _FPDFPathSegment_GetType = Module['_FPDFPathSegment_GetType'] = createExportWrapper('FPDFPathSegment_GetType', 1);
var _FPDFPathSegment_GetClose = Module['_FPDFPathSegment_GetClose'] = createExportWrapper('FPDFPathSegment_GetClose', 1);
var _FPDFPageObj_NewTextObj = Module['_FPDFPageObj_NewTextObj'] = createExportWrapper('FPDFPageObj_NewTextObj', 3);
var _FPDFText_SetText = Module['_FPDFText_SetText'] = createExportWrapper('FPDFText_SetText', 2);
var _FPDFText_SetCharcodes = Module['_FPDFText_SetCharcodes'] = createExportWrapper('FPDFText_SetCharcodes', 3);
var _FPDFText_LoadFont = Module['_FPDFText_LoadFont'] = createExportWrapper('FPDFText_LoadFont', 5);
var _FPDFText_LoadStandardFont = Module['_FPDFText_LoadStandardFont'] = createExportWrapper('FPDFText_LoadStandardFont', 2);
var _FPDFText_LoadCidType2Font = Module['_FPDFText_LoadCidType2Font'] = createExportWrapper('FPDFText_LoadCidType2Font', 6);
var _FPDFTextObj_GetFontSize = Module['_FPDFTextObj_GetFontSize'] = createExportWrapper('FPDFTextObj_GetFontSize', 2);
var _FPDFTextObj_GetText = Module['_FPDFTextObj_GetText'] = createExportWrapper('FPDFTextObj_GetText', 4);
var _FPDFTextObj_GetRenderedBitmap = Module['_FPDFTextObj_GetRenderedBitmap'] = createExportWrapper('FPDFTextObj_GetRenderedBitmap', 4);
var _FPDFFont_Close = Module['_FPDFFont_Close'] = createExportWrapper('FPDFFont_Close', 1);
var _FPDFPageObj_CreateTextObj = Module['_FPDFPageObj_CreateTextObj'] = createExportWrapper('FPDFPageObj_CreateTextObj', 3);
var _FPDFTextObj_GetTextRenderMode = Module['_FPDFTextObj_GetTextRenderMode'] = createExportWrapper('FPDFTextObj_GetTextRenderMode', 1);
var _FPDFTextObj_SetTextRenderMode = Module['_FPDFTextObj_SetTextRenderMode'] = createExportWrapper('FPDFTextObj_SetTextRenderMode', 2);
var _FPDFTextObj_GetFont = Module['_FPDFTextObj_GetFont'] = createExportWrapper('FPDFTextObj_GetFont', 1);
var _FPDFFont_GetBaseFontName = Module['_FPDFFont_GetBaseFontName'] = createExportWrapper('FPDFFont_GetBaseFontName', 3);
var _FPDFFont_GetFamilyName = Module['_FPDFFont_GetFamilyName'] = createExportWrapper('FPDFFont_GetFamilyName', 3);
var _FPDFFont_GetFontData = Module['_FPDFFont_GetFontData'] = createExportWrapper('FPDFFont_GetFontData', 4);
var _FPDFFont_GetIsEmbedded = Module['_FPDFFont_GetIsEmbedded'] = createExportWrapper('FPDFFont_GetIsEmbedded', 1);
var _FPDFFont_GetFlags = Module['_FPDFFont_GetFlags'] = createExportWrapper('FPDFFont_GetFlags', 1);
var _FPDFFont_GetWeight = Module['_FPDFFont_GetWeight'] = createExportWrapper('FPDFFont_GetWeight', 1);
var _FPDFFont_GetItalicAngle = Module['_FPDFFont_GetItalicAngle'] = createExportWrapper('FPDFFont_GetItalicAngle', 2);
var _FPDFFont_GetAscent = Module['_FPDFFont_GetAscent'] = createExportWrapper('FPDFFont_GetAscent', 3);
var _FPDFFont_GetDescent = Module['_FPDFFont_GetDescent'] = createExportWrapper('FPDFFont_GetDescent', 3);
var _FPDFFont_GetGlyphWidth = Module['_FPDFFont_GetGlyphWidth'] = createExportWrapper('FPDFFont_GetGlyphWidth', 4);
var _FPDFFont_GetGlyphPath = Module['_FPDFFont_GetGlyphPath'] = createExportWrapper('FPDFFont_GetGlyphPath', 3);
var _FPDFGlyphPath_CountGlyphSegments = Module['_FPDFGlyphPath_CountGlyphSegments'] = createExportWrapper('FPDFGlyphPath_CountGlyphSegments', 1);
var _FPDFGlyphPath_GetGlyphPathSegment = Module['_FPDFGlyphPath_GetGlyphPathSegment'] = createExportWrapper('FPDFGlyphPath_GetGlyphPathSegment', 2);
var _EPDFText_RedactInRect = Module['_EPDFText_RedactInRect'] = createExportWrapper('EPDFText_RedactInRect', 4);
var _EPDFText_RedactInQuads = Module['_EPDFText_RedactInQuads'] = createExportWrapper('EPDFText_RedactInQuads', 5);
var _FPDFDoc_GetPageMode = Module['_FPDFDoc_GetPageMode'] = createExportWrapper('FPDFDoc_GetPageMode', 1);
var _FPDFPage_Flatten = Module['_FPDFPage_Flatten'] = createExportWrapper('FPDFPage_Flatten', 2);
var _FPDFPage_HasFormFieldAtPoint = Module['_FPDFPage_HasFormFieldAtPoint'] = createExportWrapper('FPDFPage_HasFormFieldAtPoint', 4);
var _FPDFPage_FormFieldZOrderAtPoint = Module['_FPDFPage_FormFieldZOrderAtPoint'] = createExportWrapper('FPDFPage_FormFieldZOrderAtPoint', 4);
var _malloc = Module['_malloc'] = createExportWrapper('malloc', 1);
var _free = Module['_free'] = createExportWrapper('free', 1);
var _FORM_OnMouseMove = Module['_FORM_OnMouseMove'] = createExportWrapper('FORM_OnMouseMove', 5);
var _FORM_OnMouseWheel = Module['_FORM_OnMouseWheel'] = createExportWrapper('FORM_OnMouseWheel', 6);
var _FORM_OnFocus = Module['_FORM_OnFocus'] = createExportWrapper('FORM_OnFocus', 5);
var _FORM_OnLButtonDown = Module['_FORM_OnLButtonDown'] = createExportWrapper('FORM_OnLButtonDown', 5);
var _FORM_OnLButtonUp = Module['_FORM_OnLButtonUp'] = createExportWrapper('FORM_OnLButtonUp', 5);
var _FORM_OnLButtonDoubleClick = Module['_FORM_OnLButtonDoubleClick'] = createExportWrapper('FORM_OnLButtonDoubleClick', 5);
var _FORM_OnRButtonDown = Module['_FORM_OnRButtonDown'] = createExportWrapper('FORM_OnRButtonDown', 5);
var _FORM_OnRButtonUp = Module['_FORM_OnRButtonUp'] = createExportWrapper('FORM_OnRButtonUp', 5);
var _FORM_OnKeyDown = Module['_FORM_OnKeyDown'] = createExportWrapper('FORM_OnKeyDown', 4);
var _FORM_OnKeyUp = Module['_FORM_OnKeyUp'] = createExportWrapper('FORM_OnKeyUp', 4);
var _FORM_OnChar = Module['_FORM_OnChar'] = createExportWrapper('FORM_OnChar', 4);
var _FORM_GetFocusedText = Module['_FORM_GetFocusedText'] = createExportWrapper('FORM_GetFocusedText', 4);
var _FORM_GetSelectedText = Module['_FORM_GetSelectedText'] = createExportWrapper('FORM_GetSelectedText', 4);
var _FORM_ReplaceAndKeepSelection = Module['_FORM_ReplaceAndKeepSelection'] = createExportWrapper('FORM_ReplaceAndKeepSelection', 3);
var _FORM_ReplaceSelection = Module['_FORM_ReplaceSelection'] = createExportWrapper('FORM_ReplaceSelection', 3);
var _FORM_SelectAllText = Module['_FORM_SelectAllText'] = createExportWrapper('FORM_SelectAllText', 2);
var _FORM_CanUndo = Module['_FORM_CanUndo'] = createExportWrapper('FORM_CanUndo', 2);
var _FORM_CanRedo = Module['_FORM_CanRedo'] = createExportWrapper('FORM_CanRedo', 2);
var _FORM_Undo = Module['_FORM_Undo'] = createExportWrapper('FORM_Undo', 2);
var _FORM_Redo = Module['_FORM_Redo'] = createExportWrapper('FORM_Redo', 2);
var _FORM_ForceToKillFocus = Module['_FORM_ForceToKillFocus'] = createExportWrapper('FORM_ForceToKillFocus', 1);
var _FORM_GetFocusedAnnot = Module['_FORM_GetFocusedAnnot'] = createExportWrapper('FORM_GetFocusedAnnot', 3);
var _FORM_SetFocusedAnnot = Module['_FORM_SetFocusedAnnot'] = createExportWrapper('FORM_SetFocusedAnnot', 2);
var _FPDF_FFLDraw = Module['_FPDF_FFLDraw'] = createExportWrapper('FPDF_FFLDraw', 9);
var _FPDF_SetFormFieldHighlightColor = Module['_FPDF_SetFormFieldHighlightColor'] = createExportWrapper('FPDF_SetFormFieldHighlightColor', 3);
var _FPDF_SetFormFieldHighlightAlpha = Module['_FPDF_SetFormFieldHighlightAlpha'] = createExportWrapper('FPDF_SetFormFieldHighlightAlpha', 2);
var _FPDF_RemoveFormFieldHighlight = Module['_FPDF_RemoveFormFieldHighlight'] = createExportWrapper('FPDF_RemoveFormFieldHighlight', 1);
var _FORM_OnAfterLoadPage = Module['_FORM_OnAfterLoadPage'] = createExportWrapper('FORM_OnAfterLoadPage', 2);
var _FORM_OnBeforeClosePage = Module['_FORM_OnBeforeClosePage'] = createExportWrapper('FORM_OnBeforeClosePage', 2);
var _FORM_DoDocumentJSAction = Module['_FORM_DoDocumentJSAction'] = createExportWrapper('FORM_DoDocumentJSAction', 1);
var _FORM_DoDocumentOpenAction = Module['_FORM_DoDocumentOpenAction'] = createExportWrapper('FORM_DoDocumentOpenAction', 1);
var _FORM_DoDocumentAAction = Module['_FORM_DoDocumentAAction'] = createExportWrapper('FORM_DoDocumentAAction', 2);
var _FORM_DoPageAAction = Module['_FORM_DoPageAAction'] = createExportWrapper('FORM_DoPageAAction', 3);
var _FORM_SetIndexSelected = Module['_FORM_SetIndexSelected'] = createExportWrapper('FORM_SetIndexSelected', 4);
var _FORM_IsIndexSelected = Module['_FORM_IsIndexSelected'] = createExportWrapper('FORM_IsIndexSelected', 3);
var _FPDFDoc_GetJavaScriptActionCount = Module['_FPDFDoc_GetJavaScriptActionCount'] = createExportWrapper('FPDFDoc_GetJavaScriptActionCount', 1);
var _FPDFDoc_GetJavaScriptAction = Module['_FPDFDoc_GetJavaScriptAction'] = createExportWrapper('FPDFDoc_GetJavaScriptAction', 2);
var _FPDFDoc_CloseJavaScriptAction = Module['_FPDFDoc_CloseJavaScriptAction'] = createExportWrapper('FPDFDoc_CloseJavaScriptAction', 1);
var _FPDFJavaScriptAction_GetName = Module['_FPDFJavaScriptAction_GetName'] = createExportWrapper('FPDFJavaScriptAction_GetName', 3);
var _FPDFJavaScriptAction_GetScript = Module['_FPDFJavaScriptAction_GetScript'] = createExportWrapper('FPDFJavaScriptAction_GetScript', 3);
var _FPDF_ImportPagesByIndex = Module['_FPDF_ImportPagesByIndex'] = createExportWrapper('FPDF_ImportPagesByIndex', 5);
var _FPDF_ImportPages = Module['_FPDF_ImportPages'] = createExportWrapper('FPDF_ImportPages', 4);
var _FPDF_ImportNPagesToOne = Module['_FPDF_ImportNPagesToOne'] = createExportWrapper('FPDF_ImportNPagesToOne', 5);
var _FPDF_NewXObjectFromPage = Module['_FPDF_NewXObjectFromPage'] = createExportWrapper('FPDF_NewXObjectFromPage', 3);
var _FPDF_CloseXObject = Module['_FPDF_CloseXObject'] = createExportWrapper('FPDF_CloseXObject', 1);
var _FPDF_NewFormObjectFromXObject = Module['_FPDF_NewFormObjectFromXObject'] = createExportWrapper('FPDF_NewFormObjectFromXObject', 1);
var _FPDF_CopyViewerPreferences = Module['_FPDF_CopyViewerPreferences'] = createExportWrapper('FPDF_CopyViewerPreferences', 2);
var _FPDF_RenderPageBitmapWithColorScheme_Start = Module['_FPDF_RenderPageBitmapWithColorScheme_Start'] = createExportWrapper('FPDF_RenderPageBitmapWithColorScheme_Start', 10);
var _FPDF_RenderPageBitmap_Start = Module['_FPDF_RenderPageBitmap_Start'] = createExportWrapper('FPDF_RenderPageBitmap_Start', 9);
var _FPDF_RenderPage_Continue = Module['_FPDF_RenderPage_Continue'] = createExportWrapper('FPDF_RenderPage_Continue', 2);
var _FPDF_RenderPage_Close = Module['_FPDF_RenderPage_Close'] = createExportWrapper('FPDF_RenderPage_Close', 1);
var _FPDF_SaveWithVersion = Module['_FPDF_SaveWithVersion'] = createExportWrapper('FPDF_SaveWithVersion', 4);
var _FPDFText_GetCharIndexFromTextIndex = Module['_FPDFText_GetCharIndexFromTextIndex'] = createExportWrapper('FPDFText_GetCharIndexFromTextIndex', 2);
var _FPDFText_GetTextIndexFromCharIndex = Module['_FPDFText_GetTextIndexFromCharIndex'] = createExportWrapper('FPDFText_GetTextIndexFromCharIndex', 2);
var _FPDF_GetSignatureCount = Module['_FPDF_GetSignatureCount'] = createExportWrapper('FPDF_GetSignatureCount', 1);
var _FPDF_GetSignatureObject = Module['_FPDF_GetSignatureObject'] = createExportWrapper('FPDF_GetSignatureObject', 2);
var _FPDFSignatureObj_GetContents = Module['_FPDFSignatureObj_GetContents'] = createExportWrapper('FPDFSignatureObj_GetContents', 3);
var _FPDFSignatureObj_GetByteRange = Module['_FPDFSignatureObj_GetByteRange'] = createExportWrapper('FPDFSignatureObj_GetByteRange', 3);
var _FPDFSignatureObj_GetSubFilter = Module['_FPDFSignatureObj_GetSubFilter'] = createExportWrapper('FPDFSignatureObj_GetSubFilter', 3);
var _FPDFSignatureObj_GetReason = Module['_FPDFSignatureObj_GetReason'] = createExportWrapper('FPDFSignatureObj_GetReason', 3);
var _FPDFSignatureObj_GetTime = Module['_FPDFSignatureObj_GetTime'] = createExportWrapper('FPDFSignatureObj_GetTime', 3);
var _FPDFSignatureObj_GetDocMDPPermission = Module['_FPDFSignatureObj_GetDocMDPPermission'] = createExportWrapper('FPDFSignatureObj_GetDocMDPPermission', 1);
var _FPDF_StructTree_GetForPage = Module['_FPDF_StructTree_GetForPage'] = createExportWrapper('FPDF_StructTree_GetForPage', 1);
var _FPDF_StructTree_Close = Module['_FPDF_StructTree_Close'] = createExportWrapper('FPDF_StructTree_Close', 1);
var _FPDF_StructTree_CountChildren = Module['_FPDF_StructTree_CountChildren'] = createExportWrapper('FPDF_StructTree_CountChildren', 1);
var _FPDF_StructTree_GetChildAtIndex = Module['_FPDF_StructTree_GetChildAtIndex'] = createExportWrapper('FPDF_StructTree_GetChildAtIndex', 2);
var _FPDF_StructElement_GetAltText = Module['_FPDF_StructElement_GetAltText'] = createExportWrapper('FPDF_StructElement_GetAltText', 3);
var _FPDF_StructElement_GetActualText = Module['_FPDF_StructElement_GetActualText'] = createExportWrapper('FPDF_StructElement_GetActualText', 3);
var _FPDF_StructElement_GetID = Module['_FPDF_StructElement_GetID'] = createExportWrapper('FPDF_StructElement_GetID', 3);
var _FPDF_StructElement_GetLang = Module['_FPDF_StructElement_GetLang'] = createExportWrapper('FPDF_StructElement_GetLang', 3);
var _FPDF_StructElement_GetAttributeCount = Module['_FPDF_StructElement_GetAttributeCount'] = createExportWrapper('FPDF_StructElement_GetAttributeCount', 1);
var _FPDF_StructElement_GetAttributeAtIndex = Module['_FPDF_StructElement_GetAttributeAtIndex'] = createExportWrapper('FPDF_StructElement_GetAttributeAtIndex', 2);
var _FPDF_StructElement_GetStringAttribute = Module['_FPDF_StructElement_GetStringAttribute'] = createExportWrapper('FPDF_StructElement_GetStringAttribute', 4);
var _FPDF_StructElement_GetMarkedContentID = Module['_FPDF_StructElement_GetMarkedContentID'] = createExportWrapper('FPDF_StructElement_GetMarkedContentID', 1);
var _FPDF_StructElement_GetType = Module['_FPDF_StructElement_GetType'] = createExportWrapper('FPDF_StructElement_GetType', 3);
var _FPDF_StructElement_GetObjType = Module['_FPDF_StructElement_GetObjType'] = createExportWrapper('FPDF_StructElement_GetObjType', 3);
var _FPDF_StructElement_GetTitle = Module['_FPDF_StructElement_GetTitle'] = createExportWrapper('FPDF_StructElement_GetTitle', 3);
var _FPDF_StructElement_CountChildren = Module['_FPDF_StructElement_CountChildren'] = createExportWrapper('FPDF_StructElement_CountChildren', 1);
var _FPDF_StructElement_GetChildAtIndex = Module['_FPDF_StructElement_GetChildAtIndex'] = createExportWrapper('FPDF_StructElement_GetChildAtIndex', 2);
var _FPDF_StructElement_GetChildMarkedContentID = Module['_FPDF_StructElement_GetChildMarkedContentID'] = createExportWrapper('FPDF_StructElement_GetChildMarkedContentID', 2);
var _FPDF_StructElement_GetParent = Module['_FPDF_StructElement_GetParent'] = createExportWrapper('FPDF_StructElement_GetParent', 1);
var _FPDF_StructElement_Attr_GetCount = Module['_FPDF_StructElement_Attr_GetCount'] = createExportWrapper('FPDF_StructElement_Attr_GetCount', 1);
var _FPDF_StructElement_Attr_GetName = Module['_FPDF_StructElement_Attr_GetName'] = createExportWrapper('FPDF_StructElement_Attr_GetName', 5);
var _FPDF_StructElement_Attr_GetValue = Module['_FPDF_StructElement_Attr_GetValue'] = createExportWrapper('FPDF_StructElement_Attr_GetValue', 2);
var _FPDF_StructElement_Attr_GetType = Module['_FPDF_StructElement_Attr_GetType'] = createExportWrapper('FPDF_StructElement_Attr_GetType', 1);
var _FPDF_StructElement_Attr_GetBooleanValue = Module['_FPDF_StructElement_Attr_GetBooleanValue'] = createExportWrapper('FPDF_StructElement_Attr_GetBooleanValue', 2);
var _FPDF_StructElement_Attr_GetNumberValue = Module['_FPDF_StructElement_Attr_GetNumberValue'] = createExportWrapper('FPDF_StructElement_Attr_GetNumberValue', 2);
var _FPDF_StructElement_Attr_GetStringValue = Module['_FPDF_StructElement_Attr_GetStringValue'] = createExportWrapper('FPDF_StructElement_Attr_GetStringValue', 4);
var _FPDF_StructElement_Attr_GetBlobValue = Module['_FPDF_StructElement_Attr_GetBlobValue'] = createExportWrapper('FPDF_StructElement_Attr_GetBlobValue', 4);
var _FPDF_StructElement_Attr_CountChildren = Module['_FPDF_StructElement_Attr_CountChildren'] = createExportWrapper('FPDF_StructElement_Attr_CountChildren', 1);
var _FPDF_StructElement_Attr_GetChildAtIndex = Module['_FPDF_StructElement_Attr_GetChildAtIndex'] = createExportWrapper('FPDF_StructElement_Attr_GetChildAtIndex', 2);
var _FPDF_StructElement_GetMarkedContentIdCount = Module['_FPDF_StructElement_GetMarkedContentIdCount'] = createExportWrapper('FPDF_StructElement_GetMarkedContentIdCount', 1);
var _FPDF_StructElement_GetMarkedContentIdAtIndex = Module['_FPDF_StructElement_GetMarkedContentIdAtIndex'] = createExportWrapper('FPDF_StructElement_GetMarkedContentIdAtIndex', 2);
var _FPDF_AddInstalledFont = Module['_FPDF_AddInstalledFont'] = createExportWrapper('FPDF_AddInstalledFont', 3);
var _FPDF_SetSystemFontInfo = Module['_FPDF_SetSystemFontInfo'] = createExportWrapper('FPDF_SetSystemFontInfo', 1);
var _FPDF_GetDefaultTTFMap = Module['_FPDF_GetDefaultTTFMap'] = createExportWrapper('FPDF_GetDefaultTTFMap', 0);
var _FPDF_GetDefaultTTFMapCount = Module['_FPDF_GetDefaultTTFMapCount'] = createExportWrapper('FPDF_GetDefaultTTFMapCount', 0);
var _FPDF_GetDefaultTTFMapEntry = Module['_FPDF_GetDefaultTTFMapEntry'] = createExportWrapper('FPDF_GetDefaultTTFMapEntry', 1);
var _FPDF_GetDefaultSystemFontInfo = Module['_FPDF_GetDefaultSystemFontInfo'] = createExportWrapper('FPDF_GetDefaultSystemFontInfo', 0);
var _FPDF_FreeDefaultSystemFontInfo = Module['_FPDF_FreeDefaultSystemFontInfo'] = createExportWrapper('FPDF_FreeDefaultSystemFontInfo', 1);
var _FPDFText_LoadPage = Module['_FPDFText_LoadPage'] = createExportWrapper('FPDFText_LoadPage', 1);
var _FPDFText_ClosePage = Module['_FPDFText_ClosePage'] = createExportWrapper('FPDFText_ClosePage', 1);
var _FPDFText_CountChars = Module['_FPDFText_CountChars'] = createExportWrapper('FPDFText_CountChars', 1);
var _FPDFText_GetUnicode = Module['_FPDFText_GetUnicode'] = createExportWrapper('FPDFText_GetUnicode', 2);
var _FPDFText_GetTextObject = Module['_FPDFText_GetTextObject'] = createExportWrapper('FPDFText_GetTextObject', 2);
var _FPDFText_IsGenerated = Module['_FPDFText_IsGenerated'] = createExportWrapper('FPDFText_IsGenerated', 2);
var _FPDFText_IsHyphen = Module['_FPDFText_IsHyphen'] = createExportWrapper('FPDFText_IsHyphen', 2);
var _FPDFText_HasUnicodeMapError = Module['_FPDFText_HasUnicodeMapError'] = createExportWrapper('FPDFText_HasUnicodeMapError', 2);
var _FPDFText_GetFontSize = Module['_FPDFText_GetFontSize'] = createExportWrapper('FPDFText_GetFontSize', 2);
var _FPDFText_GetFontInfo = Module['_FPDFText_GetFontInfo'] = createExportWrapper('FPDFText_GetFontInfo', 5);
var _FPDFText_GetFontWeight = Module['_FPDFText_GetFontWeight'] = createExportWrapper('FPDFText_GetFontWeight', 2);
var _FPDFText_GetFillColor = Module['_FPDFText_GetFillColor'] = createExportWrapper('FPDFText_GetFillColor', 6);
var _FPDFText_GetStrokeColor = Module['_FPDFText_GetStrokeColor'] = createExportWrapper('FPDFText_GetStrokeColor', 6);
var _FPDFText_GetCharAngle = Module['_FPDFText_GetCharAngle'] = createExportWrapper('FPDFText_GetCharAngle', 2);
var _FPDFText_GetCharBox = Module['_FPDFText_GetCharBox'] = createExportWrapper('FPDFText_GetCharBox', 6);
var _FPDFText_GetLooseCharBox = Module['_FPDFText_GetLooseCharBox'] = createExportWrapper('FPDFText_GetLooseCharBox', 3);
var _FPDFText_GetMatrix = Module['_FPDFText_GetMatrix'] = createExportWrapper('FPDFText_GetMatrix', 3);
var _FPDFText_GetCharOrigin = Module['_FPDFText_GetCharOrigin'] = createExportWrapper('FPDFText_GetCharOrigin', 4);
var _FPDFText_GetCharIndexAtPos = Module['_FPDFText_GetCharIndexAtPos'] = createExportWrapper('FPDFText_GetCharIndexAtPos', 5);
var _FPDFText_GetText = Module['_FPDFText_GetText'] = createExportWrapper('FPDFText_GetText', 4);
var _FPDFText_CountRects = Module['_FPDFText_CountRects'] = createExportWrapper('FPDFText_CountRects', 3);
var _FPDFText_GetRect = Module['_FPDFText_GetRect'] = createExportWrapper('FPDFText_GetRect', 6);
var _FPDFText_GetBoundedText = Module['_FPDFText_GetBoundedText'] = createExportWrapper('FPDFText_GetBoundedText', 7);
var _FPDFText_FindStart = Module['_FPDFText_FindStart'] = createExportWrapper('FPDFText_FindStart', 4);
var _FPDFText_FindNext = Module['_FPDFText_FindNext'] = createExportWrapper('FPDFText_FindNext', 1);
var _FPDFText_FindPrev = Module['_FPDFText_FindPrev'] = createExportWrapper('FPDFText_FindPrev', 1);
var _FPDFText_GetSchResultIndex = Module['_FPDFText_GetSchResultIndex'] = createExportWrapper('FPDFText_GetSchResultIndex', 1);
var _FPDFText_GetSchCount = Module['_FPDFText_GetSchCount'] = createExportWrapper('FPDFText_GetSchCount', 1);
var _FPDFText_FindClose = Module['_FPDFText_FindClose'] = createExportWrapper('FPDFText_FindClose', 1);
var _FPDFLink_LoadWebLinks = Module['_FPDFLink_LoadWebLinks'] = createExportWrapper('FPDFLink_LoadWebLinks', 1);
var _FPDFLink_CountWebLinks = Module['_FPDFLink_CountWebLinks'] = createExportWrapper('FPDFLink_CountWebLinks', 1);
var _FPDFLink_GetURL = Module['_FPDFLink_GetURL'] = createExportWrapper('FPDFLink_GetURL', 4);
var _FPDFLink_CountRects = Module['_FPDFLink_CountRects'] = createExportWrapper('FPDFLink_CountRects', 2);
var _FPDFLink_GetRect = Module['_FPDFLink_GetRect'] = createExportWrapper('FPDFLink_GetRect', 7);
var _FPDFLink_GetTextRange = Module['_FPDFLink_GetTextRange'] = createExportWrapper('FPDFLink_GetTextRange', 4);
var _FPDFLink_CloseWebLinks = Module['_FPDFLink_CloseWebLinks'] = createExportWrapper('FPDFLink_CloseWebLinks', 1);
var _FPDFPage_GetDecodedThumbnailData = Module['_FPDFPage_GetDecodedThumbnailData'] = createExportWrapper('FPDFPage_GetDecodedThumbnailData', 3);
var _FPDFPage_GetRawThumbnailData = Module['_FPDFPage_GetRawThumbnailData'] = createExportWrapper('FPDFPage_GetRawThumbnailData', 3);
var _FPDFPage_GetThumbnailAsBitmap = Module['_FPDFPage_GetThumbnailAsBitmap'] = createExportWrapper('FPDFPage_GetThumbnailAsBitmap', 1);
var _FPDFPage_SetMediaBox = Module['_FPDFPage_SetMediaBox'] = createExportWrapper('FPDFPage_SetMediaBox', 5);
var _FPDFPage_SetCropBox = Module['_FPDFPage_SetCropBox'] = createExportWrapper('FPDFPage_SetCropBox', 5);
var _FPDFPage_SetBleedBox = Module['_FPDFPage_SetBleedBox'] = createExportWrapper('FPDFPage_SetBleedBox', 5);
var _FPDFPage_SetTrimBox = Module['_FPDFPage_SetTrimBox'] = createExportWrapper('FPDFPage_SetTrimBox', 5);
var _FPDFPage_SetArtBox = Module['_FPDFPage_SetArtBox'] = createExportWrapper('FPDFPage_SetArtBox', 5);
var _FPDFPage_GetMediaBox = Module['_FPDFPage_GetMediaBox'] = createExportWrapper('FPDFPage_GetMediaBox', 5);
var _FPDFPage_GetCropBox = Module['_FPDFPage_GetCropBox'] = createExportWrapper('FPDFPage_GetCropBox', 5);
var _FPDFPage_GetBleedBox = Module['_FPDFPage_GetBleedBox'] = createExportWrapper('FPDFPage_GetBleedBox', 5);
var _FPDFPage_GetTrimBox = Module['_FPDFPage_GetTrimBox'] = createExportWrapper('FPDFPage_GetTrimBox', 5);
var _FPDFPage_GetArtBox = Module['_FPDFPage_GetArtBox'] = createExportWrapper('FPDFPage_GetArtBox', 5);
var _FPDFPage_TransFormWithClip = Module['_FPDFPage_TransFormWithClip'] = createExportWrapper('FPDFPage_TransFormWithClip', 3);
var _FPDFPageObj_TransformClipPath = Module['_FPDFPageObj_TransformClipPath'] = createExportWrapper('FPDFPageObj_TransformClipPath', 7);
var _FPDFPageObj_GetClipPath = Module['_FPDFPageObj_GetClipPath'] = createExportWrapper('FPDFPageObj_GetClipPath', 1);
var _FPDFClipPath_CountPaths = Module['_FPDFClipPath_CountPaths'] = createExportWrapper('FPDFClipPath_CountPaths', 1);
var _FPDFClipPath_CountPathSegments = Module['_FPDFClipPath_CountPathSegments'] = createExportWrapper('FPDFClipPath_CountPathSegments', 2);
var _FPDFClipPath_GetPathSegment = Module['_FPDFClipPath_GetPathSegment'] = createExportWrapper('FPDFClipPath_GetPathSegment', 3);
var _FPDF_CreateClipPath = Module['_FPDF_CreateClipPath'] = createExportWrapper('FPDF_CreateClipPath', 4);
var _FPDF_DestroyClipPath = Module['_FPDF_DestroyClipPath'] = createExportWrapper('FPDF_DestroyClipPath', 1);
var _FPDFPage_InsertClipPath = Module['_FPDFPage_InsertClipPath'] = createExportWrapper('FPDFPage_InsertClipPath', 2);
var _FPDF_InitLibrary = Module['_FPDF_InitLibrary'] = createExportWrapper('FPDF_InitLibrary', 0);
var _FPDF_DestroyLibrary = Module['_FPDF_DestroyLibrary'] = createExportWrapper('FPDF_DestroyLibrary', 0);
var _FPDF_SetSandBoxPolicy = Module['_FPDF_SetSandBoxPolicy'] = createExportWrapper('FPDF_SetSandBoxPolicy', 2);
var _FPDF_LoadDocument = Module['_FPDF_LoadDocument'] = createExportWrapper('FPDF_LoadDocument', 2);
var _FPDF_GetFormType = Module['_FPDF_GetFormType'] = createExportWrapper('FPDF_GetFormType', 1);
var _FPDF_LoadXFA = Module['_FPDF_LoadXFA'] = createExportWrapper('FPDF_LoadXFA', 1);
var _FPDF_LoadMemDocument = Module['_FPDF_LoadMemDocument'] = createExportWrapper('FPDF_LoadMemDocument', 3);
var _FPDF_LoadMemDocument64 = Module['_FPDF_LoadMemDocument64'] = createExportWrapper('FPDF_LoadMemDocument64', 3);
var _FPDF_LoadCustomDocument = Module['_FPDF_LoadCustomDocument'] = createExportWrapper('FPDF_LoadCustomDocument', 2);
var _FPDF_GetFileVersion = Module['_FPDF_GetFileVersion'] = createExportWrapper('FPDF_GetFileVersion', 2);
var _FPDF_DocumentHasValidCrossReferenceTable = Module['_FPDF_DocumentHasValidCrossReferenceTable'] = createExportWrapper('FPDF_DocumentHasValidCrossReferenceTable', 1);
var _FPDF_GetDocPermissions = Module['_FPDF_GetDocPermissions'] = createExportWrapper('FPDF_GetDocPermissions', 1);
var _FPDF_GetDocUserPermissions = Module['_FPDF_GetDocUserPermissions'] = createExportWrapper('FPDF_GetDocUserPermissions', 1);
var _FPDF_GetSecurityHandlerRevision = Module['_FPDF_GetSecurityHandlerRevision'] = createExportWrapper('FPDF_GetSecurityHandlerRevision', 1);
var _FPDF_GetPageCount = Module['_FPDF_GetPageCount'] = createExportWrapper('FPDF_GetPageCount', 1);
var _FPDF_LoadPage = Module['_FPDF_LoadPage'] = createExportWrapper('FPDF_LoadPage', 2);
var _FPDF_GetPageWidthF = Module['_FPDF_GetPageWidthF'] = createExportWrapper('FPDF_GetPageWidthF', 1);
var _FPDF_GetPageWidth = Module['_FPDF_GetPageWidth'] = createExportWrapper('FPDF_GetPageWidth', 1);
var _FPDF_GetPageHeightF = Module['_FPDF_GetPageHeightF'] = createExportWrapper('FPDF_GetPageHeightF', 1);
var _FPDF_GetPageHeight = Module['_FPDF_GetPageHeight'] = createExportWrapper('FPDF_GetPageHeight', 1);
var _FPDF_GetPageBoundingBox = Module['_FPDF_GetPageBoundingBox'] = createExportWrapper('FPDF_GetPageBoundingBox', 2);
var _FPDF_RenderPageBitmap = Module['_FPDF_RenderPageBitmap'] = createExportWrapper('FPDF_RenderPageBitmap', 8);
var _FPDF_RenderPageBitmapWithMatrix = Module['_FPDF_RenderPageBitmapWithMatrix'] = createExportWrapper('FPDF_RenderPageBitmapWithMatrix', 5);
var _EPDF_RenderAnnotBitmap = Module['_EPDF_RenderAnnotBitmap'] = createExportWrapper('EPDF_RenderAnnotBitmap', 6);
var _FPDF_ClosePage = Module['_FPDF_ClosePage'] = createExportWrapper('FPDF_ClosePage', 1);
var _FPDF_CloseDocument = Module['_FPDF_CloseDocument'] = createExportWrapper('FPDF_CloseDocument', 1);
var _FPDF_GetLastError = Module['_FPDF_GetLastError'] = createExportWrapper('FPDF_GetLastError', 0);
var _FPDF_DeviceToPage = Module['_FPDF_DeviceToPage'] = createExportWrapper('FPDF_DeviceToPage', 10);
var _FPDF_PageToDevice = Module['_FPDF_PageToDevice'] = createExportWrapper('FPDF_PageToDevice', 10);
var _FPDFBitmap_Create = Module['_FPDFBitmap_Create'] = createExportWrapper('FPDFBitmap_Create', 3);
var _FPDFBitmap_CreateEx = Module['_FPDFBitmap_CreateEx'] = createExportWrapper('FPDFBitmap_CreateEx', 5);
var _FPDFBitmap_GetFormat = Module['_FPDFBitmap_GetFormat'] = createExportWrapper('FPDFBitmap_GetFormat', 1);
var _FPDFBitmap_FillRect = Module['_FPDFBitmap_FillRect'] = createExportWrapper('FPDFBitmap_FillRect', 6);
var _FPDFBitmap_GetBuffer = Module['_FPDFBitmap_GetBuffer'] = createExportWrapper('FPDFBitmap_GetBuffer', 1);
var _FPDFBitmap_GetWidth = Module['_FPDFBitmap_GetWidth'] = createExportWrapper('FPDFBitmap_GetWidth', 1);
var _FPDFBitmap_GetHeight = Module['_FPDFBitmap_GetHeight'] = createExportWrapper('FPDFBitmap_GetHeight', 1);
var _FPDFBitmap_GetStride = Module['_FPDFBitmap_GetStride'] = createExportWrapper('FPDFBitmap_GetStride', 1);
var _FPDFBitmap_Destroy = Module['_FPDFBitmap_Destroy'] = createExportWrapper('FPDFBitmap_Destroy', 1);
var _FPDF_GetPageSizeByIndexF = Module['_FPDF_GetPageSizeByIndexF'] = createExportWrapper('FPDF_GetPageSizeByIndexF', 3);
var _EPDF_GetPageRotationByIndex = Module['_EPDF_GetPageRotationByIndex'] = createExportWrapper('EPDF_GetPageRotationByIndex', 2);
var _FPDF_GetPageSizeByIndex = Module['_FPDF_GetPageSizeByIndex'] = createExportWrapper('FPDF_GetPageSizeByIndex', 4);
var _FPDF_VIEWERREF_GetPrintScaling = Module['_FPDF_VIEWERREF_GetPrintScaling'] = createExportWrapper('FPDF_VIEWERREF_GetPrintScaling', 1);
var _FPDF_VIEWERREF_GetNumCopies = Module['_FPDF_VIEWERREF_GetNumCopies'] = createExportWrapper('FPDF_VIEWERREF_GetNumCopies', 1);
var _FPDF_VIEWERREF_GetPrintPageRange = Module['_FPDF_VIEWERREF_GetPrintPageRange'] = createExportWrapper('FPDF_VIEWERREF_GetPrintPageRange', 1);
var _FPDF_VIEWERREF_GetPrintPageRangeCount = Module['_FPDF_VIEWERREF_GetPrintPageRangeCount'] = createExportWrapper('FPDF_VIEWERREF_GetPrintPageRangeCount', 1);
var _FPDF_VIEWERREF_GetPrintPageRangeElement = Module['_FPDF_VIEWERREF_GetPrintPageRangeElement'] = createExportWrapper('FPDF_VIEWERREF_GetPrintPageRangeElement', 2);
var _FPDF_VIEWERREF_GetDuplex = Module['_FPDF_VIEWERREF_GetDuplex'] = createExportWrapper('FPDF_VIEWERREF_GetDuplex', 1);
var _FPDF_VIEWERREF_GetName = Module['_FPDF_VIEWERREF_GetName'] = createExportWrapper('FPDF_VIEWERREF_GetName', 4);
var _FPDF_CountNamedDests = Module['_FPDF_CountNamedDests'] = createExportWrapper('FPDF_CountNamedDests', 1);
var _FPDF_GetNamedDestByName = Module['_FPDF_GetNamedDestByName'] = createExportWrapper('FPDF_GetNamedDestByName', 2);
var _FPDF_GetNamedDest = Module['_FPDF_GetNamedDest'] = createExportWrapper('FPDF_GetNamedDest', 4);
var _FPDF_GetXFAPacketCount = Module['_FPDF_GetXFAPacketCount'] = createExportWrapper('FPDF_GetXFAPacketCount', 1);
var _FPDF_GetXFAPacketName = Module['_FPDF_GetXFAPacketName'] = createExportWrapper('FPDF_GetXFAPacketName', 4);
var _FPDF_GetXFAPacketContent = Module['_FPDF_GetXFAPacketContent'] = createExportWrapper('FPDF_GetXFAPacketContent', 5);
var _FPDF_GetTrailerEnds = Module['_FPDF_GetTrailerEnds'] = createExportWrapper('FPDF_GetTrailerEnds', 3);
var _fflush = createExportWrapper('fflush', 1);
var _emscripten_builtin_memalign = createExportWrapper('emscripten_builtin_memalign', 2);
var _strerror = createExportWrapper('strerror', 1);
var _setThrew = createExportWrapper('setThrew', 2);
var __emscripten_tempret_set = createExportWrapper('_emscripten_tempret_set', 1);
var _emscripten_stack_init = () => (_emscripten_stack_init = wasmExports['emscripten_stack_init'])();
var _emscripten_stack_get_free = () => (_emscripten_stack_get_free = wasmExports['emscripten_stack_get_free'])();
var _emscripten_stack_get_base = () => (_emscripten_stack_get_base = wasmExports['emscripten_stack_get_base'])();
var _emscripten_stack_get_end = () => (_emscripten_stack_get_end = wasmExports['emscripten_stack_get_end'])();
var __emscripten_stack_restore = (a0) => (__emscripten_stack_restore = wasmExports['_emscripten_stack_restore'])(a0);
var __emscripten_stack_alloc = (a0) => (__emscripten_stack_alloc = wasmExports['_emscripten_stack_alloc'])(a0);
var _emscripten_stack_get_current = () => (_emscripten_stack_get_current = wasmExports['emscripten_stack_get_current'])();
var dynCall_ji = Module['dynCall_ji'] = createExportWrapper('dynCall_ji', 2);
var dynCall_jij = Module['dynCall_jij'] = createExportWrapper('dynCall_jij', 4);
var dynCall_iiij = Module['dynCall_iiij'] = createExportWrapper('dynCall_iiij', 5);
var dynCall_iij = Module['dynCall_iij'] = createExportWrapper('dynCall_iij', 4);
var dynCall_j = Module['dynCall_j'] = createExportWrapper('dynCall_j', 1);
var dynCall_jji = Module['dynCall_jji'] = createExportWrapper('dynCall_jji', 4);
var dynCall_iji = Module['dynCall_iji'] = createExportWrapper('dynCall_iji', 4);
var dynCall_viijii = Module['dynCall_viijii'] = createExportWrapper('dynCall_viijii', 7);
var dynCall_iiji = Module['dynCall_iiji'] = createExportWrapper('dynCall_iiji', 5);
var dynCall_jiji = Module['dynCall_jiji'] = createExportWrapper('dynCall_jiji', 5);
var dynCall_iiiiij = Module['dynCall_iiiiij'] = createExportWrapper('dynCall_iiiiij', 7);
var dynCall_iiiiijj = Module['dynCall_iiiiijj'] = createExportWrapper('dynCall_iiiiijj', 9);
var dynCall_iiiiiijj = Module['dynCall_iiiiiijj'] = createExportWrapper('dynCall_iiiiiijj', 10);
var dynCall_viji = Module['dynCall_viji'] = createExportWrapper('dynCall_viji', 5);

function invoke_viii(index,a1,a2,a3) {
  var sp = stackSave();
  try {
    getWasmTableEntry(index)(a1,a2,a3);
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}

function invoke_ii(index,a1) {
  var sp = stackSave();
  try {
    return getWasmTableEntry(index)(a1);
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}

function invoke_iii(index,a1,a2) {
  var sp = stackSave();
  try {
    return getWasmTableEntry(index)(a1,a2);
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}

function invoke_iiii(index,a1,a2,a3) {
  var sp = stackSave();
  try {
    return getWasmTableEntry(index)(a1,a2,a3);
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}

function invoke_viiii(index,a1,a2,a3,a4) {
  var sp = stackSave();
  try {
    getWasmTableEntry(index)(a1,a2,a3,a4);
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}

function invoke_iiiii(index,a1,a2,a3,a4) {
  var sp = stackSave();
  try {
    return getWasmTableEntry(index)(a1,a2,a3,a4);
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}

function invoke_v(index) {
  var sp = stackSave();
  try {
    getWasmTableEntry(index)();
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}

function invoke_vii(index,a1,a2) {
  var sp = stackSave();
  try {
    getWasmTableEntry(index)(a1,a2);
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}

function invoke_viiiiiiiii(index,a1,a2,a3,a4,a5,a6,a7,a8,a9) {
  var sp = stackSave();
  try {
    getWasmTableEntry(index)(a1,a2,a3,a4,a5,a6,a7,a8,a9);
  } catch(e) {
    stackRestore(sp);
    if (e !== e+0) throw e;
    _setThrew(1, 0);
  }
}


// include: postamble.js
// === Auto-generated postamble setup entry stuff ===

Module['wasmExports'] = wasmExports;
Module['ccall'] = ccall;
Module['cwrap'] = cwrap;
Module['addFunction'] = addFunction;
Module['removeFunction'] = removeFunction;
Module['setValue'] = setValue;
Module['getValue'] = getValue;
Module['UTF8ToString'] = UTF8ToString;
Module['stringToUTF8'] = stringToUTF8;
Module['UTF16ToString'] = UTF16ToString;
Module['stringToUTF16'] = stringToUTF16;
var missingLibrarySymbols = [
  'writeI53ToI64',
  'writeI53ToI64Clamped',
  'writeI53ToI64Signaling',
  'writeI53ToU64Clamped',
  'writeI53ToU64Signaling',
  'readI53FromI64',
  'readI53FromU64',
  'convertI32PairToI53',
  'convertU32PairToI53',
  'getTempRet0',
  'setTempRet0',
  'exitJS',
  'inetPton4',
  'inetNtop4',
  'inetPton6',
  'inetNtop6',
  'readSockaddr',
  'writeSockaddr',
  'emscriptenLog',
  'readEmAsmArgs',
  'jstoi_q',
  'listenOnce',
  'autoResumeAudioContext',
  'dynCallLegacy',
  'getDynCaller',
  'dynCall',
  'handleException',
  'keepRuntimeAlive',
  'runtimeKeepalivePush',
  'runtimeKeepalivePop',
  'callUserCallback',
  'maybeExit',
  'asmjsMangle',
  'HandleAllocator',
  'getNativeTypeSize',
  'STACK_SIZE',
  'STACK_ALIGN',
  'POINTER_SIZE',
  'ASSERTIONS',
  'reallyNegative',
  'unSign',
  'strLen',
  'reSign',
  'formatString',
  'intArrayToString',
  'AsciiToString',
  'lengthBytesUTF16',
  'UTF32ToString',
  'stringToUTF32',
  'lengthBytesUTF32',
  'stringToNewUTF8',
  'registerKeyEventCallback',
  'maybeCStringToJsString',
  'findEventTarget',
  'getBoundingClientRect',
  'fillMouseEventData',
  'registerMouseEventCallback',
  'registerWheelEventCallback',
  'registerUiEventCallback',
  'registerFocusEventCallback',
  'fillDeviceOrientationEventData',
  'registerDeviceOrientationEventCallback',
  'fillDeviceMotionEventData',
  'registerDeviceMotionEventCallback',
  'screenOrientation',
  'fillOrientationChangeEventData',
  'registerOrientationChangeEventCallback',
  'fillFullscreenChangeEventData',
  'registerFullscreenChangeEventCallback',
  'JSEvents_requestFullscreen',
  'JSEvents_resizeCanvasForFullscreen',
  'registerRestoreOldStyle',
  'hideEverythingExceptGivenElement',
  'restoreHiddenElements',
  'setLetterbox',
  'softFullscreenResizeWebGLRenderTarget',
  'doRequestFullscreen',
  'fillPointerlockChangeEventData',
  'registerPointerlockChangeEventCallback',
  'registerPointerlockErrorEventCallback',
  'requestPointerLock',
  'fillVisibilityChangeEventData',
  'registerVisibilityChangeEventCallback',
  'registerTouchEventCallback',
  'fillGamepadEventData',
  'registerGamepadEventCallback',
  'registerBeforeUnloadEventCallback',
  'fillBatteryEventData',
  'battery',
  'registerBatteryEventCallback',
  'setCanvasElementSize',
  'getCanvasElementSize',
  'jsStackTrace',
  'getCallstack',
  'convertPCtoSourceLocation',
  'checkWasiClock',
  'wasiRightsToMuslOFlags',
  'wasiOFlagsToMuslOFlags',
  'createDyncallWrapper',
  'safeSetTimeout',
  'setImmediateWrapped',
  'clearImmediateWrapped',
  'polyfillSetImmediate',
  'registerPostMainLoop',
  'registerPreMainLoop',
  'getPromise',
  'makePromise',
  'idsToPromises',
  'makePromiseCallback',
  'ExceptionInfo',
  'findMatchingCatch',
  'Browser_asyncPrepareDataCounter',
  'safeRequestAnimationFrame',
  'arraySum',
  'addDays',
  'getSocketFromFD',
  'getSocketAddress',
  'FS_unlink',
  'FS_mkdirTree',
  '_setNetworkCallback',
  'heapObjectForWebGLType',
  'toTypedArrayIndex',
  'webgl_enable_ANGLE_instanced_arrays',
  'webgl_enable_OES_vertex_array_object',
  'webgl_enable_WEBGL_draw_buffers',
  'webgl_enable_WEBGL_multi_draw',
  'webgl_enable_EXT_polygon_offset_clamp',
  'webgl_enable_EXT_clip_control',
  'webgl_enable_WEBGL_polygon_mode',
  'emscriptenWebGLGet',
  'computeUnpackAlignedImageSize',
  'colorChannelsInGlTextureFormat',
  'emscriptenWebGLGetTexPixelData',
  'emscriptenWebGLGetUniform',
  'webglGetUniformLocation',
  'webglPrepareUniformLocationsBeforeFirstUse',
  'webglGetLeftBracePos',
  'emscriptenWebGLGetVertexAttrib',
  '__glGetActiveAttribOrUniform',
  'writeGLArray',
  'registerWebGlEventCallback',
  'runAndAbortIfError',
  'ALLOC_NORMAL',
  'ALLOC_STACK',
  'allocate',
  'writeStringToMemory',
  'writeAsciiToMemory',
  'setErrNo',
  'demangle',
  'stackTrace',
];
missingLibrarySymbols.forEach(missingLibrarySymbol)

var unexportedSymbols = [
  'run',
  'addOnPreRun',
  'addOnInit',
  'addOnPreMain',
  'addOnExit',
  'addOnPostRun',
  'addRunDependency',
  'removeRunDependency',
  'out',
  'err',
  'callMain',
  'abort',
  'wasmMemory',
  'writeStackCookie',
  'checkStackCookie',
  'convertI32PairToI53Checked',
  'stackSave',
  'stackRestore',
  'stackAlloc',
  'ptrToString',
  'zeroMemory',
  'getHeapMax',
  'growMemory',
  'ENV',
  'ERRNO_CODES',
  'strError',
  'DNS',
  'Protocols',
  'Sockets',
  'initRandomFill',
  'randomFill',
  'timers',
  'warnOnce',
  'readEmAsmArgsArray',
  'jstoi_s',
  'getExecutableName',
  'asyncLoad',
  'alignMemory',
  'mmapAlloc',
  'wasmTable',
  'noExitRuntime',
  'getCFunc',
  'uleb128Encode',
  'sigToWasmTypes',
  'generateFuncType',
  'convertJsFunctionToWasm',
  'freeTableIndexes',
  'functionsInTableMap',
  'getEmptyTableSlot',
  'updateTableMap',
  'getFunctionAddress',
  'PATH',
  'PATH_FS',
  'UTF8Decoder',
  'UTF8ArrayToString',
  'stringToUTF8Array',
  'lengthBytesUTF8',
  'intArrayFromString',
  'stringToAscii',
  'UTF16Decoder',
  'stringToUTF8OnStack',
  'writeArrayToMemory',
  'JSEvents',
  'specialHTMLTargets',
  'findCanvasEventTarget',
  'currentFullscreenStrategy',
  'restoreOldWindowedStyle',
  'UNWIND_CACHE',
  'ExitStatus',
  'getEnvStrings',
  'doReadv',
  'doWritev',
  'promiseMap',
  'uncaughtExceptionCount',
  'exceptionLast',
  'exceptionCaught',
  'Browser',
  'getPreloadedImageData__data',
  'wget',
  'MONTH_DAYS_REGULAR',
  'MONTH_DAYS_LEAP',
  'MONTH_DAYS_REGULAR_CUMULATIVE',
  'MONTH_DAYS_LEAP_CUMULATIVE',
  'isLeapYear',
  'ydayFromDate',
  'SYSCALLS',
  'preloadPlugins',
  'FS_createPreloadedFile',
  'FS_modeStringToFlags',
  'FS_getMode',
  'FS_stdin_getChar_buffer',
  'FS_stdin_getChar',
  'FS_createPath',
  'FS_createDevice',
  'FS_readFile',
  'FS',
  'FS_createDataFile',
  'FS_createLazyFile',
  'MEMFS',
  'TTY',
  'PIPEFS',
  'SOCKFS',
  'tempFixedLengthArray',
  'miniTempWebGLFloatBuffers',
  'miniTempWebGLIntBuffers',
  'GL',
  'AL',
  'GLUT',
  'EGL',
  'GLEW',
  'IDBStore',
  'SDL',
  'SDL_gfx',
  'allocateUTF8',
  'allocateUTF8OnStack',
  'print',
  'printErr',
];
unexportedSymbols.forEach(unexportedRuntimeSymbol);



var calledRun;
var calledPrerun;

dependenciesFulfilled = function runCaller() {
  // If run has never been called, and we should call run (INVOKE_RUN is true, and Module.noInitialRun is not false)
  if (!calledRun) run();
  if (!calledRun) dependenciesFulfilled = runCaller; // try this again later, after new deps are fulfilled
};

function stackCheckInit() {
  // This is normally called automatically during __wasm_call_ctors but need to
  // get these values before even running any of the ctors so we call it redundantly
  // here.
  _emscripten_stack_init();
  // TODO(sbc): Move writeStackCookie to native to to avoid this.
  writeStackCookie();
}

function run() {

  if (runDependencies > 0) {
    return;
  }

  stackCheckInit();

  if (!calledPrerun) {
    calledPrerun = 1;
    preRun();

    // a preRun added a dependency, run will be called later
    if (runDependencies > 0) {
      return;
    }
  }

  function doRun() {
    // run may have just been called through dependencies being fulfilled just in this very frame,
    // or while the async setStatus time below was happening
    if (calledRun) return;
    calledRun = 1;
    Module['calledRun'] = 1;

    if (ABORT) return;

    initRuntime();

    readyPromiseResolve(Module);
    Module['onRuntimeInitialized']?.();

    assert(!Module['_main'], 'compiled without a main, but one is present. if you added it from JS, use Module["onRuntimeInitialized"]');

    postRun();
  }

  if (Module['setStatus']) {
    Module['setStatus']('Running...');
    setTimeout(() => {
      setTimeout(() => Module['setStatus'](''), 1);
      doRun();
    }, 1);
  } else
  {
    doRun();
  }
  checkStackCookie();
}

function checkUnflushedContent() {
  // Compiler settings do not allow exiting the runtime, so flushing
  // the streams is not possible. but in ASSERTIONS mode we check
  // if there was something to flush, and if so tell the user they
  // should request that the runtime be exitable.
  // Normally we would not even include flush() at all, but in ASSERTIONS
  // builds we do so just for this check, and here we see if there is any
  // content to flush, that is, we check if there would have been
  // something a non-ASSERTIONS build would have not seen.
  // How we flush the streams depends on whether we are in SYSCALLS_REQUIRE_FILESYSTEM=0
  // mode (which has its own special function for this; otherwise, all
  // the code is inside libc)
  var oldOut = out;
  var oldErr = err;
  var has = false;
  out = err = (x) => {
    has = true;
  }
  try { // it doesn't matter if it fails
    _fflush(0);
    // also flush in the JS FS layer
    ['stdout', 'stderr'].forEach((name) => {
      var info = FS.analyzePath('/dev/' + name);
      if (!info) return;
      var stream = info.object;
      var rdev = stream.rdev;
      var tty = TTY.ttys[rdev];
      if (tty?.output?.length) {
        has = true;
      }
    });
  } catch(e) {}
  out = oldOut;
  err = oldErr;
  if (has) {
    warnOnce('stdio streams had content in them that was not flushed. you should set EXIT_RUNTIME to 1 (see the Emscripten FAQ), or make sure to emit a newline when you printf etc.');
  }
}

if (Module['preInit']) {
  if (typeof Module['preInit'] == 'function') Module['preInit'] = [Module['preInit']];
  while (Module['preInit'].length > 0) {
    Module['preInit'].pop()();
  }
}

run();

// end include: postamble.js

// include: postamble_modularize.js
// In MODULARIZE mode we wrap the generated code in a factory function
// and return either the Module itself, or a promise of the module.
//
// We assign to the `moduleRtn` global here and configure closure to see
// this as and extern so it won't get minified.

moduleRtn = readyPromise;

// Assertion for attempting to access module properties on the incoming
// moduleArg.  In the past we used this object as the prototype of the module
// and assigned properties to it, but now we return a distinct object.  This
// keeps the instance private until it is ready (i.e the promise has been
// resolved).
for (const prop of Object.keys(Module)) {
  if (!(prop in moduleArg)) {
    Object.defineProperty(moduleArg, prop, {
      configurable: true,
      get() {
        abort(`Access to module property ('${prop}') is no longer possible via the module constructor argument; Instead, use the result of the module constructor.`)
      }
    });
  }
}
// end include: postamble_modularize.js



  return moduleRtn;
}
);
})();
if (typeof exports === 'object' && typeof module === 'object')
  module.exports = createPdfium;
else if (typeof define === 'function' && define['amd'])
  define([], () => createPdfium);

==== FILE: C:\Users\charris\Downloads\embed-pdf-viewer-main\embed-pdf-viewer-main\packages\pdfium\src\vendor\pdfium.d.cts ====
/// <reference types="emscripten" />

export interface PdfiumModule extends EmscriptenModule {}

/**
 * Create an instance of pdfium wasm module
 * @param init - override pdfium methods
 */
export default function createPdfium<T>(init: Partial<PdfiumModule>): Promise<PdfiumModule & T>;

==== FILE: C:\Users\charris\Downloads\embed-pdf-viewer-main\embed-pdf-viewer-main\packages\pdfium\src\vendor\pdfium.d.ts ====
/// <reference types="emscripten" />

export interface PdfiumModule extends EmscriptenModule {}

/**
 * Create an instance of pdfium wasm module
 * @param init - override pdfium methods
 */
export default function createPdfium<T>(init: Partial<PdfiumModule>): Promise<PdfiumModule & T>;

==== FILE: C:\Users\charris\Downloads\embed-pdf-viewer-main\embed-pdf-viewer-main\packages\pdfium\src\vendor\pdfium.js ====
var createPdfium = (() => {
  var _scriptName = import.meta.url;

  return async function (moduleArg = {}) {
    var moduleRtn;

    // include: shell.js
    // The Module object: Our interface to the outside world. We import
    // and export values on it. There are various ways Module can be used:
    // 1. Not defined. We create it here
    // 2. A function parameter, function(moduleArg) => Promise<Module>
    // 3. pre-run appended it, var Module = {}; ..generated code..
    // 4. External script tag defines var Module.
    // We need to check if Module already exists (e.g. case 3 above).
    // Substitution will be replaced with actual code on later stage of the build,
    // this way Closure Compiler will not mangle it (e.g. case 4. above).
    // Note that if you want to run closure, and also to use Module
    // after the generated code, you will need to define   var Module = {};
    // before the code. Then that object will be used in the code, and you
    // can continue to use Module afterwards as well.
    var Module = moduleArg;

    // Set up the promise that indicates the Module is initialized
    var readyPromiseResolve, readyPromiseReject;
    var readyPromise = new Promise((resolve, reject) => {
      readyPromiseResolve = resolve;
      readyPromiseReject = reject;
    });
    [
      '_EPDF_GetMetaKeyCount',
      '_EPDF_GetMetaKeyName',
      '_EPDF_GetMetaTrapped',
      '_EPDF_GetPageRotationByIndex',
      '_EPDF_HasMetaText',
      '_EPDF_PNG_EncodeRGBA',
      '_EPDF_RenderAnnotBitmap',
      '_EPDF_SetMetaText',
      '_EPDF_SetMetaTrapped',
      '_EPDFAction_CreateGoTo',
      '_EPDFAction_CreateGoToNamed',
      '_EPDFAction_CreateLaunch',
      '_EPDFAction_CreateRemoteGoToByName',
      '_EPDFAction_CreateRemoteGoToDest',
      '_EPDFAction_CreateURI',
      '_EPDFAnnot_ClearColor',
      '_EPDFAnnot_GenerateAppearance',
      '_EPDFAnnot_GenerateAppearanceWithBlend',
      '_EPDFAnnot_GetBlendMode',
      '_EPDFAnnot_GetBorderDashPattern',
      '_EPDFAnnot_GetBorderDashPatternCount',
      '_EPDFAnnot_GetBorderEffect',
      '_EPDFAnnot_GetBorderStyle',
      '_EPDFAnnot_GetColor',
      '_EPDFAnnot_GetDefaultAppearance',
      '_EPDFAnnot_GetIcon',
      '_EPDFAnnot_GetIntent',
      '_EPDFAnnot_GetLineEndings',
      '_EPDFAnnot_GetOpacity',
      '_EPDFAnnot_GetRectangleDifferences',
      '_EPDFAnnot_GetRichContent',
      '_EPDFAnnot_GetTextAlignment',
      '_EPDFAnnot_GetVerticalAlignment',
      '_EPDFAnnot_SetBorderDashPattern',
      '_EPDFAnnot_SetBorderStyle',
      '_EPDFAnnot_SetColor',
      '_EPDFAnnot_SetDefaultAppearance',
      '_EPDFAnnot_SetIcon',
      '_EPDFAnnot_SetIntent',
      '_EPDFAnnot_SetLine',
      '_EPDFAnnot_SetLineEndings',
      '_EPDFAnnot_SetLinkedAnnot',
      '_EPDFAnnot_SetOpacity',
      '_EPDFAnnot_SetTextAlignment',
      '_EPDFAnnot_SetVerticalAlignment',
      '_EPDFAnnot_SetVertices',
      '_EPDFAnnot_UpdateAppearanceToRect',
      '_EPDFAttachment_GetDescription',
      '_EPDFAttachment_GetIntegerValue',
      '_EPDFAttachment_SetDescription',
      '_EPDFAttachment_SetSubtype',
      '_EPDFBookmark_AppendChild',
      '_EPDFBookmark_Clear',
      '_EPDFBookmark_ClearTarget',
      '_EPDFBookmark_Create',
      '_EPDFBookmark_Delete',
      '_EPDFBookmark_InsertAfter',
      '_EPDFBookmark_SetAction',
      '_EPDFBookmark_SetDest',
      '_EPDFBookmark_SetTitle',
      '_EPDFCatalog_GetLanguage',
      '_EPDFDest_CreateRemoteView',
      '_EPDFDest_CreateRemoteXYZ',
      '_EPDFDest_CreateView',
      '_EPDFDest_CreateXYZ',
      '_EPDFNamedDest_Remove',
      '_EPDFNamedDest_SetDest',
      '_EPDFPage_CreateAnnot',
      '_EPDFPage_GetAnnotByName',
      '_EPDFPage_GetAnnotCountRaw',
      '_EPDFPage_GetAnnotRaw',
      '_EPDFPage_RemoveAnnotByName',
      '_EPDFPage_RemoveAnnotRaw',
      '_EPDFText_RedactInQuads',
      '_EPDFText_RedactInRect',
      '_FORM_CanRedo',
      '_FORM_CanUndo',
      '_FORM_DoDocumentAAction',
      '_FORM_DoDocumentJSAction',
      '_FORM_DoDocumentOpenAction',
      '_FORM_DoPageAAction',
      '_FORM_ForceToKillFocus',
      '_FORM_GetFocusedAnnot',
      '_FORM_GetFocusedText',
      '_FORM_GetSelectedText',
      '_FORM_IsIndexSelected',
      '_FORM_OnAfterLoadPage',
      '_FORM_OnBeforeClosePage',
      '_FORM_OnChar',
      '_FORM_OnFocus',
      '_FORM_OnKeyDown',
      '_FORM_OnKeyUp',
      '_FORM_OnLButtonDoubleClick',
      '_FORM_OnLButtonDown',
      '_FORM_OnLButtonUp',
      '_FORM_OnMouseMove',
      '_FORM_OnMouseWheel',
      '_FORM_OnRButtonDown',
      '_FORM_OnRButtonUp',
      '_FORM_Redo',
      '_FORM_ReplaceAndKeepSelection',
      '_FORM_ReplaceSelection',
      '_FORM_SelectAllText',
      '_FORM_SetFocusedAnnot',
      '_FORM_SetIndexSelected',
      '_FORM_Undo',
      '_FPDF_AddInstalledFont',
      '_FPDF_CloseDocument',
      '_FPDF_ClosePage',
      '_FPDF_CloseXObject',
      '_FPDF_CopyViewerPreferences',
      '_FPDF_CountNamedDests',
      '_FPDF_CreateClipPath',
      '_FPDF_CreateNewDocument',
      '_FPDF_DestroyClipPath',
      '_FPDF_DestroyLibrary',
      '_FPDF_DeviceToPage',
      '_FPDF_DocumentHasValidCrossReferenceTable',
      '_FPDF_FFLDraw',
      '_FPDF_FreeDefaultSystemFontInfo',
      '_FPDF_GetDefaultSystemFontInfo',
      '_FPDF_GetDefaultTTFMap',
      '_FPDF_GetDefaultTTFMapCount',
      '_FPDF_GetDefaultTTFMapEntry',
      '_FPDF_GetDocPermissions',
      '_FPDF_GetDocUserPermissions',
      '_FPDF_GetFileIdentifier',
      '_FPDF_GetFileVersion',
      '_FPDF_GetFormType',
      '_FPDF_GetLastError',
      '_FPDF_GetMetaText',
      '_FPDF_GetNamedDest',
      '_FPDF_GetNamedDestByName',
      '_FPDF_GetPageAAction',
      '_FPDF_GetPageBoundingBox',
      '_FPDF_GetPageCount',
      '_FPDF_GetPageHeight',
      '_FPDF_GetPageHeightF',
      '_FPDF_GetPageLabel',
      '_FPDF_GetPageSizeByIndex',
      '_FPDF_GetPageSizeByIndexF',
      '_FPDF_GetPageWidth',
      '_FPDF_GetPageWidthF',
      '_FPDF_GetSecurityHandlerRevision',
      '_FPDF_GetSignatureCount',
      '_FPDF_GetSignatureObject',
      '_FPDF_GetTrailerEnds',
      '_FPDF_GetXFAPacketContent',
      '_FPDF_GetXFAPacketCount',
      '_FPDF_GetXFAPacketName',
      '_FPDF_ImportNPagesToOne',
      '_FPDF_ImportPages',
      '_FPDF_ImportPagesByIndex',
      '_FPDF_InitLibrary',
      '_FPDF_InitLibraryWithConfig',
      '_FPDF_LoadCustomDocument',
      '_FPDF_LoadDocument',
      '_FPDF_LoadMemDocument',
      '_FPDF_LoadMemDocument64',
      '_FPDF_LoadPage',
      '_FPDF_LoadXFA',
      '_FPDF_MovePages',
      '_FPDF_NewFormObjectFromXObject',
      '_FPDF_NewXObjectFromPage',
      '_FPDF_PageToDevice',
      '_FPDF_RemoveFormFieldHighlight',
      '_FPDF_RenderPage_Close',
      '_FPDF_RenderPage_Continue',
      '_FPDF_RenderPageBitmap',
      '_FPDF_RenderPageBitmap_Start',
      '_FPDF_RenderPageBitmapWithColorScheme_Start',
      '_FPDF_RenderPageBitmapWithMatrix',
      '_FPDF_SaveAsCopy',
      '_FPDF_SaveWithVersion',
      '_FPDF_SetFormFieldHighlightAlpha',
      '_FPDF_SetFormFieldHighlightColor',
      '_FPDF_SetSandBoxPolicy',
      '_FPDF_SetSystemFontInfo',
      '_FPDF_StructElement_Attr_CountChildren',
      '_FPDF_StructElement_Attr_GetBlobValue',
      '_FPDF_StructElement_Attr_GetBooleanValue',
      '_FPDF_StructElement_Attr_GetChildAtIndex',
      '_FPDF_StructElement_Attr_GetCount',
      '_FPDF_StructElement_Attr_GetName',
      '_FPDF_StructElement_Attr_GetNumberValue',
      '_FPDF_StructElement_Attr_GetStringValue',
      '_FPDF_StructElement_Attr_GetType',
      '_FPDF_StructElement_Attr_GetValue',
      '_FPDF_StructElement_CountChildren',
      '_FPDF_StructElement_GetActualText',
      '_FPDF_StructElement_GetAltText',
      '_FPDF_StructElement_GetAttributeAtIndex',
      '_FPDF_StructElement_GetAttributeCount',
      '_FPDF_StructElement_GetChildAtIndex',
      '_FPDF_StructElement_GetChildMarkedContentID',
      '_FPDF_StructElement_GetID',
      '_FPDF_StructElement_GetLang',
      '_FPDF_StructElement_GetMarkedContentID',
      '_FPDF_StructElement_GetMarkedContentIdAtIndex',
      '_FPDF_StructElement_GetMarkedContentIdCount',
      '_FPDF_StructElement_GetObjType',
      '_FPDF_StructElement_GetParent',
      '_FPDF_StructElement_GetStringAttribute',
      '_FPDF_StructElement_GetTitle',
      '_FPDF_StructElement_GetType',
      '_FPDF_StructTree_Close',
      '_FPDF_StructTree_CountChildren',
      '_FPDF_StructTree_GetChildAtIndex',
      '_FPDF_StructTree_GetForPage',
      '_FPDF_VIEWERREF_GetDuplex',
      '_FPDF_VIEWERREF_GetName',
      '_FPDF_VIEWERREF_GetNumCopies',
      '_FPDF_VIEWERREF_GetPrintPageRange',
      '_FPDF_VIEWERREF_GetPrintPageRangeCount',
      '_FPDF_VIEWERREF_GetPrintPageRangeElement',
      '_FPDF_VIEWERREF_GetPrintScaling',
      '_FPDFAction_GetDest',
      '_FPDFAction_GetFilePath',
      '_FPDFAction_GetType',
      '_FPDFAction_GetURIPath',
      '_FPDFAnnot_AddFileAttachment',
      '_FPDFAnnot_AddInkStroke',
      '_FPDFAnnot_AppendAttachmentPoints',
      '_FPDFAnnot_AppendObject',
      '_FPDFAnnot_CountAttachmentPoints',
      '_FPDFAnnot_GetAP',
      '_FPDFAnnot_GetAttachmentPoints',
      '_FPDFAnnot_GetBorder',
      '_FPDFAnnot_GetColor',
      '_FPDFAnnot_GetFileAttachment',
      '_FPDFAnnot_GetFlags',
      '_FPDFAnnot_GetFocusableSubtypes',
      '_FPDFAnnot_GetFocusableSubtypesCount',
      '_FPDFAnnot_GetFontColor',
      '_FPDFAnnot_GetFontSize',
      '_FPDFAnnot_GetFormAdditionalActionJavaScript',
      '_FPDFAnnot_GetFormControlCount',
      '_FPDFAnnot_GetFormControlIndex',
      '_FPDFAnnot_GetFormFieldAlternateName',
      '_FPDFAnnot_GetFormFieldAtPoint',
      '_FPDFAnnot_GetFormFieldExportValue',
      '_FPDFAnnot_GetFormFieldFlags',
      '_FPDFAnnot_GetFormFieldName',
      '_FPDFAnnot_GetFormFieldType',
      '_FPDFAnnot_GetFormFieldValue',
      '_FPDFAnnot_GetInkListCount',
      '_FPDFAnnot_GetInkListPath',
      '_FPDFAnnot_GetLine',
      '_FPDFAnnot_GetLink',
      '_FPDFAnnot_GetLinkedAnnot',
      '_FPDFAnnot_GetNumberValue',
      '_FPDFAnnot_GetObject',
      '_FPDFAnnot_GetObjectCount',
      '_FPDFAnnot_GetOptionCount',
      '_FPDFAnnot_GetOptionLabel',
      '_FPDFAnnot_GetRect',
      '_FPDFAnnot_GetStringValue',
      '_FPDFAnnot_GetSubtype',
      '_FPDFAnnot_GetValueType',
      '_FPDFAnnot_GetVertices',
      '_FPDFAnnot_HasAttachmentPoints',
      '_FPDFAnnot_HasKey',
      '_FPDFAnnot_IsChecked',
      '_FPDFAnnot_IsObjectSupportedSubtype',
      '_FPDFAnnot_IsOptionSelected',
      '_FPDFAnnot_IsSupportedSubtype',
      '_FPDFAnnot_RemoveInkList',
      '_FPDFAnnot_RemoveObject',
      '_FPDFAnnot_SetAP',
      '_FPDFAnnot_SetAttachmentPoints',
      '_FPDFAnnot_SetBorder',
      '_FPDFAnnot_SetColor',
      '_FPDFAnnot_SetFlags',
      '_FPDFAnnot_SetFocusableSubtypes',
      '_FPDFAnnot_SetFontColor',
      '_FPDFAnnot_SetFormFieldFlags',
      '_FPDFAnnot_SetRect',
      '_FPDFAnnot_SetStringValue',
      '_FPDFAnnot_SetURI',
      '_FPDFAnnot_UpdateObject',
      '_FPDFAttachment_GetFile',
      '_FPDFAttachment_GetName',
      '_FPDFAttachment_GetStringValue',
      '_FPDFAttachment_GetSubtype',
      '_FPDFAttachment_GetValueType',
      '_FPDFAttachment_HasKey',
      '_FPDFAttachment_SetFile',
      '_FPDFAttachment_SetStringValue',
      '_FPDFAvail_Create',
      '_FPDFAvail_Destroy',
      '_FPDFAvail_GetDocument',
      '_FPDFAvail_GetFirstPageNum',
      '_FPDFAvail_IsDocAvail',
      '_FPDFAvail_IsFormAvail',
      '_FPDFAvail_IsLinearized',
      '_FPDFAvail_IsPageAvail',
      '_FPDFBitmap_Create',
      '_FPDFBitmap_CreateEx',
      '_FPDFBitmap_Destroy',
      '_FPDFBitmap_FillRect',
      '_FPDFBitmap_GetBuffer',
      '_FPDFBitmap_GetFormat',
      '_FPDFBitmap_GetHeight',
      '_FPDFBitmap_GetStride',
      '_FPDFBitmap_GetWidth',
      '_FPDFBookmark_Find',
      '_FPDFBookmark_GetAction',
      '_FPDFBookmark_GetCount',
      '_FPDFBookmark_GetDest',
      '_FPDFBookmark_GetFirstChild',
      '_FPDFBookmark_GetNextSibling',
      '_FPDFBookmark_GetTitle',
      '_FPDFCatalog_IsTagged',
      '_FPDFCatalog_SetLanguage',
      '_FPDFClipPath_CountPaths',
      '_FPDFClipPath_CountPathSegments',
      '_FPDFClipPath_GetPathSegment',
      '_FPDFDest_GetDestPageIndex',
      '_FPDFDest_GetLocationInPage',
      '_FPDFDest_GetView',
      '_FPDFDoc_AddAttachment',
      '_FPDFDoc_CloseJavaScriptAction',
      '_FPDFDoc_DeleteAttachment',
      '_FPDFDOC_ExitFormFillEnvironment',
      '_FPDFDoc_GetAttachment',
      '_FPDFDoc_GetAttachmentCount',
      '_FPDFDoc_GetJavaScriptAction',
      '_FPDFDoc_GetJavaScriptActionCount',
      '_FPDFDoc_GetPageMode',
      '_FPDFDOC_InitFormFillEnvironment',
      '_FPDFFont_Close',
      '_FPDFFont_GetAscent',
      '_FPDFFont_GetBaseFontName',
      '_FPDFFont_GetDescent',
      '_FPDFFont_GetFamilyName',
      '_FPDFFont_GetFlags',
      '_FPDFFont_GetFontData',
      '_FPDFFont_GetGlyphPath',
      '_FPDFFont_GetGlyphWidth',
      '_FPDFFont_GetIsEmbedded',
      '_FPDFFont_GetItalicAngle',
      '_FPDFFont_GetWeight',
      '_FPDFFormObj_CountObjects',
      '_FPDFFormObj_GetObject',
      '_FPDFFormObj_RemoveObject',
      '_FPDFGlyphPath_CountGlyphSegments',
      '_FPDFGlyphPath_GetGlyphPathSegment',
      '_FPDFImageObj_GetBitmap',
      '_FPDFImageObj_GetIccProfileDataDecoded',
      '_FPDFImageObj_GetImageDataDecoded',
      '_FPDFImageObj_GetImageDataRaw',
      '_FPDFImageObj_GetImageFilter',
      '_FPDFImageObj_GetImageFilterCount',
      '_FPDFImageObj_GetImageMetadata',
      '_FPDFImageObj_GetImagePixelSize',
      '_FPDFImageObj_GetRenderedBitmap',
      '_FPDFImageObj_LoadJpegFile',
      '_FPDFImageObj_LoadJpegFileInline',
      '_FPDFImageObj_SetBitmap',
      '_FPDFImageObj_SetMatrix',
      '_FPDFJavaScriptAction_GetName',
      '_FPDFJavaScriptAction_GetScript',
      '_FPDFLink_CloseWebLinks',
      '_FPDFLink_CountQuadPoints',
      '_FPDFLink_CountRects',
      '_FPDFLink_CountWebLinks',
      '_FPDFLink_Enumerate',
      '_FPDFLink_GetAction',
      '_FPDFLink_GetAnnot',
      '_FPDFLink_GetAnnotRect',
      '_FPDFLink_GetDest',
      '_FPDFLink_GetLinkAtPoint',
      '_FPDFLink_GetLinkZOrderAtPoint',
      '_FPDFLink_GetQuadPoints',
      '_FPDFLink_GetRect',
      '_FPDFLink_GetTextRange',
      '_FPDFLink_GetURL',
      '_FPDFLink_LoadWebLinks',
      '_FPDFPage_CloseAnnot',
      '_FPDFPage_CountObjects',
      '_FPDFPage_CreateAnnot',
      '_FPDFPage_Delete',
      '_FPDFPage_Flatten',
      '_FPDFPage_FormFieldZOrderAtPoint',
      '_FPDFPage_GenerateContent',
      '_FPDFPage_GetAnnot',
      '_FPDFPage_GetAnnotCount',
      '_FPDFPage_GetAnnotIndex',
      '_FPDFPage_GetArtBox',
      '_FPDFPage_GetBleedBox',
      '_FPDFPage_GetCropBox',
      '_FPDFPage_GetDecodedThumbnailData',
      '_FPDFPage_GetMediaBox',
      '_FPDFPage_GetObject',
      '_FPDFPage_GetRawThumbnailData',
      '_FPDFPage_GetRotation',
      '_FPDFPage_GetThumbnailAsBitmap',
      '_FPDFPage_GetTrimBox',
      '_FPDFPage_HasFormFieldAtPoint',
      '_FPDFPage_HasTransparency',
      '_FPDFPage_InsertClipPath',
      '_FPDFPage_InsertObject',
      '_FPDFPage_InsertObjectAtIndex',
      '_FPDFPage_New',
      '_FPDFPage_RemoveAnnot',
      '_FPDFPage_RemoveObject',
      '_FPDFPage_SetArtBox',
      '_FPDFPage_SetBleedBox',
      '_FPDFPage_SetCropBox',
      '_FPDFPage_SetMediaBox',
      '_FPDFPage_SetRotation',
      '_FPDFPage_SetTrimBox',
      '_FPDFPage_TransformAnnots',
      '_FPDFPage_TransFormWithClip',
      '_FPDFPageObj_AddMark',
      '_FPDFPageObj_CountMarks',
      '_FPDFPageObj_CreateNewPath',
      '_FPDFPageObj_CreateNewRect',
      '_FPDFPageObj_CreateTextObj',
      '_FPDFPageObj_Destroy',
      '_FPDFPageObj_GetBounds',
      '_FPDFPageObj_GetClipPath',
      '_FPDFPageObj_GetDashArray',
      '_FPDFPageObj_GetDashCount',
      '_FPDFPageObj_GetDashPhase',
      '_FPDFPageObj_GetFillColor',
      '_FPDFPageObj_GetIsActive',
      '_FPDFPageObj_GetLineCap',
      '_FPDFPageObj_GetLineJoin',
      '_FPDFPageObj_GetMark',
      '_FPDFPageObj_GetMarkedContentID',
      '_FPDFPageObj_GetMatrix',
      '_FPDFPageObj_GetRotatedBounds',
      '_FPDFPageObj_GetStrokeColor',
      '_FPDFPageObj_GetStrokeWidth',
      '_FPDFPageObj_GetType',
      '_FPDFPageObj_HasTransparency',
      '_FPDFPageObj_NewImageObj',
      '_FPDFPageObj_NewTextObj',
      '_FPDFPageObj_RemoveMark',
      '_FPDFPageObj_SetBlendMode',
      '_FPDFPageObj_SetDashArray',
      '_FPDFPageObj_SetDashPhase',
      '_FPDFPageObj_SetFillColor',
      '_FPDFPageObj_SetIsActive',
      '_FPDFPageObj_SetLineCap',
      '_FPDFPageObj_SetLineJoin',
      '_FPDFPageObj_SetMatrix',
      '_FPDFPageObj_SetStrokeColor',
      '_FPDFPageObj_SetStrokeWidth',
      '_FPDFPageObj_Transform',
      '_FPDFPageObj_TransformClipPath',
      '_FPDFPageObj_TransformF',
      '_FPDFPageObjMark_CountParams',
      '_FPDFPageObjMark_GetName',
      '_FPDFPageObjMark_GetParamBlobValue',
      '_FPDFPageObjMark_GetParamIntValue',
      '_FPDFPageObjMark_GetParamKey',
      '_FPDFPageObjMark_GetParamStringValue',
      '_FPDFPageObjMark_GetParamValueType',
      '_FPDFPageObjMark_RemoveParam',
      '_FPDFPageObjMark_SetBlobParam',
      '_FPDFPageObjMark_SetIntParam',
      '_FPDFPageObjMark_SetStringParam',
      '_FPDFPath_BezierTo',
      '_FPDFPath_Close',
      '_FPDFPath_CountSegments',
      '_FPDFPath_GetDrawMode',
      '_FPDFPath_GetPathSegment',
      '_FPDFPath_LineTo',
      '_FPDFPath_MoveTo',
      '_FPDFPath_SetDrawMode',
      '_FPDFPathSegment_GetClose',
      '_FPDFPathSegment_GetPoint',
      '_FPDFPathSegment_GetType',
      '_FPDFSignatureObj_GetByteRange',
      '_FPDFSignatureObj_GetContents',
      '_FPDFSignatureObj_GetDocMDPPermission',
      '_FPDFSignatureObj_GetReason',
      '_FPDFSignatureObj_GetSubFilter',
      '_FPDFSignatureObj_GetTime',
      '_FPDFText_ClosePage',
      '_FPDFText_CountChars',
      '_FPDFText_CountRects',
      '_FPDFText_FindClose',
      '_FPDFText_FindNext',
      '_FPDFText_FindPrev',
      '_FPDFText_FindStart',
      '_FPDFText_GetBoundedText',
      '_FPDFText_GetCharAngle',
      '_FPDFText_GetCharBox',
      '_FPDFText_GetCharIndexAtPos',
      '_FPDFText_GetCharIndexFromTextIndex',
      '_FPDFText_GetCharOrigin',
      '_FPDFText_GetFillColor',
      '_FPDFText_GetFontInfo',
      '_FPDFText_GetFontSize',
      '_FPDFText_GetFontWeight',
      '_FPDFText_GetLooseCharBox',
      '_FPDFText_GetMatrix',
      '_FPDFText_GetRect',
      '_FPDFText_GetSchCount',
      '_FPDFText_GetSchResultIndex',
      '_FPDFText_GetStrokeColor',
      '_FPDFText_GetText',
      '_FPDFText_GetTextIndexFromCharIndex',
      '_FPDFText_GetTextObject',
      '_FPDFText_GetUnicode',
      '_FPDFText_HasUnicodeMapError',
      '_FPDFText_IsGenerated',
      '_FPDFText_IsHyphen',
      '_FPDFText_LoadCidType2Font',
      '_FPDFText_LoadFont',
      '_FPDFText_LoadPage',
      '_FPDFText_LoadStandardFont',
      '_FPDFText_SetCharcodes',
      '_FPDFText_SetText',
      '_FPDFTextObj_GetFont',
      '_FPDFTextObj_GetFontSize',
      '_FPDFTextObj_GetRenderedBitmap',
      '_FPDFTextObj_GetText',
      '_FPDFTextObj_GetTextRenderMode',
      '_FPDFTextObj_SetTextRenderMode',
      '_PDFiumExt_CloseFileWriter',
      '_PDFiumExt_CloseFormFillInfo',
      '_PDFiumExt_ExitFormFillEnvironment',
      '_PDFiumExt_GetFileWriterData',
      '_PDFiumExt_GetFileWriterSize',
      '_PDFiumExt_Init',
      '_PDFiumExt_InitFormFillEnvironment',
      '_PDFiumExt_OpenFileWriter',
      '_PDFiumExt_OpenFormFillInfo',
      '_PDFiumExt_SaveAsCopy',
      '_malloc',
      '_free',
      '_memory',
      '___indirect_function_table',
      'onRuntimeInitialized',
    ].forEach((prop) => {
      if (!Object.getOwnPropertyDescriptor(readyPromise, prop)) {
        Object.defineProperty(readyPromise, prop, {
          get: () =>
            abort(
              'You are getting ' +
                prop +
                ' on the Promise object, instead of the instance. Use .then() to get called back with the instance, see the MODULARIZE docs in src/settings.js',
            ),
          set: () =>
            abort(
              'You are setting ' +
                prop +
                ' on the Promise object, instead of the instance. Use .then() to get called back with the instance, see the MODULARIZE docs in src/settings.js',
            ),
        });
      }
    });

    // Determine the runtime environment we are in. You can customize this by
    // setting the ENVIRONMENT setting at compile time (see settings.js).

    // Attempt to auto-detect the environment
    var ENVIRONMENT_IS_WEB = typeof window == 'object';
    var ENVIRONMENT_IS_WORKER = typeof importScripts == 'function';
    // N.b. Electron.js environment is simultaneously a NODE-environment, but
    // also a web environment.
    var ENVIRONMENT_IS_NODE =
      typeof process == 'object' &&
      typeof process.versions == 'object' &&
      typeof process.versions.node == 'string' &&
      process.type != 'renderer';
    var ENVIRONMENT_IS_SHELL =
      !ENVIRONMENT_IS_WEB && !ENVIRONMENT_IS_NODE && !ENVIRONMENT_IS_WORKER;

    if (ENVIRONMENT_IS_NODE) {
      // `require()` is no-op in an ESM module, use `createRequire()` to construct
      // the require()` function.  This is only necessary for multi-environment
      // builds, `-sENVIRONMENT=node` emits a static import declaration instead.
      // TODO: Swap all `require()`'s with `import()`'s?
      const { createRequire } = await import('module');
      let dirname = import.meta.url;
      if (dirname.startsWith('data:')) {
        dirname = '/';
      }
      /** @suppress{duplicate} */
      var require = createRequire(dirname);
    }

    // --pre-jses are emitted after the Module integration code, so that they can
    // refer to Module (if they choose; they can also define Module)

    // Sometimes an existing Module object exists with properties
    // meant to overwrite the default module functionality. Here
    // we collect those properties and reapply _after_ we configure
    // the current environment's defaults to avoid having to be so
    // defensive during initialization.
    var moduleOverrides = Object.assign({}, Module);

    var arguments_ = [];
    var thisProgram = './this.program';
    var quit_ = (status, toThrow) => {
      throw toThrow;
    };

    // `/` should be present at the end if `scriptDirectory` is not empty
    var scriptDirectory = '';
    function locateFile(path) {
      if (Module['locateFile']) {
        return Module['locateFile'](path, scriptDirectory);
      }
      return scriptDirectory + path;
    }

    // Hooks that are implemented differently in different runtime environments.
    var readAsync, readBinary;

    if (ENVIRONMENT_IS_NODE) {
      if (typeof process == 'undefined' || !process.release || process.release.name !== 'node')
        throw new Error(
          'not compiled for this environment (did you build to HTML and try to run it not on the web, or set ENVIRONMENT to something - like node - and run it someplace else - like on the web?)',
        );

      var nodeVersion = process.versions.node;
      var numericVersion = nodeVersion.split('.').slice(0, 3);
      numericVersion =
        numericVersion[0] * 10000 + numericVersion[1] * 100 + numericVersion[2].split('-')[0] * 1;
      var minVersion = 160000;
      if (numericVersion < 160000) {
        throw new Error(
          'This emscripten-generated code requires node v16.0.0 (detected v' + nodeVersion + ')',
        );
      }

      // These modules will usually be used on Node.js. Load them eagerly to avoid
      // the complexity of lazy-loading.
      var fs = require('fs');
      var nodePath = require('path');

      // EXPORT_ES6 + ENVIRONMENT_IS_NODE always requires use of import.meta.url,
      // since there's no way getting the current absolute path of the module when
      // support for that is not available.
      if (!import.meta.url.startsWith('data:')) {
        scriptDirectory = nodePath.dirname(require('url').fileURLToPath(import.meta.url)) + '/';
      }

      // include: node_shell_read.js
      readBinary = (filename) => {
        // We need to re-wrap `file://` strings to URLs. Normalizing isn't
        // necessary in that case, the path should already be absolute.
        filename = isFileURI(filename) ? new URL(filename) : nodePath.normalize(filename);
        var ret = fs.readFileSync(filename);
        assert(ret.buffer);
        return ret;
      };

      readAsync = (filename, binary = true) => {
        // See the comment in the `readBinary` function.
        filename = isFileURI(filename) ? new URL(filename) : nodePath.normalize(filename);
        return new Promise((resolve, reject) => {
          fs.readFile(filename, binary ? undefined : 'utf8', (err, data) => {
            if (err) reject(err);
            else resolve(binary ? data.buffer : data);
          });
        });
      };
      // end include: node_shell_read.js
      if (!Module['thisProgram'] && process.argv.length > 1) {
        thisProgram = process.argv[1].replace(/\\/g, '/');
      }

      arguments_ = process.argv.slice(2);

      // MODULARIZE will export the module in the proper place outside, we don't need to export here

      quit_ = (status, toThrow) => {
        process.exitCode = status;
        throw toThrow;
      };
    } else if (ENVIRONMENT_IS_SHELL) {
      if (
        (typeof process == 'object' && typeof require === 'function') ||
        typeof window == 'object' ||
        typeof importScripts == 'function'
      )
        throw new Error(
          'not compiled for this environment (did you build to HTML and try to run it not on the web, or set ENVIRONMENT to something - like node - and run it someplace else - like on the web?)',
        );

      readBinary = (f) => {
        if (typeof readbuffer == 'function') {
          return new Uint8Array(readbuffer(f));
        }
        let data = read(f, 'binary');
        assert(typeof data == 'object');
        return data;
      };

      readAsync = (f) => {
        return new Promise((resolve, reject) => {
          setTimeout(() => resolve(readBinary(f)));
        });
      };

      globalThis.clearTimeout ??= (id) => {};

      // spidermonkey lacks setTimeout but we use it above in readAsync.
      globalThis.setTimeout ??= (f) => (typeof f == 'function' ? f() : abort());

      // v8 uses `arguments_` whereas spidermonkey uses `scriptArgs`
      arguments_ = globalThis.arguments || globalThis.scriptArgs;

      if (typeof quit == 'function') {
        quit_ = (status, toThrow) => {
          // Unlike node which has process.exitCode, d8 has no such mechanism. So we
          // have no way to set the exit code and then let the program exit with
          // that code when it naturally stops running (say, when all setTimeouts
          // have completed). For that reason, we must call `quit` - the only way to
          // set the exit code - but quit also halts immediately.  To increase
          // consistency with node (and the web) we schedule the actual quit call
          // using a setTimeout to give the current stack and any exception handlers
          // a chance to run.  This enables features such as addOnPostRun (which
          // expected to be able to run code after main returns).
          setTimeout(() => {
            if (!(toThrow instanceof ExitStatus)) {
              let toLog = toThrow;
              if (toThrow && typeof toThrow == 'object' && toThrow.stack) {
                toLog = [toThrow, toThrow.stack];
              }
              err(`exiting due to exception: ${toLog}`);
            }
            quit(status);
          });
          throw toThrow;
        };
      }

      if (typeof print != 'undefined') {
        // Prefer to use print/printErr where they exist, as they usually work better.
        globalThis.console ??= /** @type{!Console} */ ({});
        console.log = /** @type{!function(this:Console, ...*): undefined} */ (print);
        console.warn = console.error = /** @type{!function(this:Console, ...*): undefined} */ (
          globalThis.printErr ?? print
        );
      }
    } else if (ENVIRONMENT_IS_WEB || ENVIRONMENT_IS_WORKER) {
      // Note that this includes Node.js workers when relevant (pthreads is enabled).
      // Node.js workers are detected as a combination of ENVIRONMENT_IS_WORKER and
      // ENVIRONMENT_IS_NODE.
      if (ENVIRONMENT_IS_WORKER) {
        // Check worker, not web, since window could be polyfilled
        scriptDirectory = self.location.href;
      } else if (typeof document != 'undefined' && document.currentScript) {
        // web
        scriptDirectory = document.currentScript.src;
      }
      // When MODULARIZE, this JS may be executed later, after document.currentScript
      // is gone, so we saved it, and we use it here instead of any other info.
      if (_scriptName) {
        scriptDirectory = _scriptName;
      }
      // blob urls look like blob:http://site.com/etc/etc and we cannot infer anything from them.
      // otherwise, slice off the final part of the url to find the script directory.
      // if scriptDirectory does not contain a slash, lastIndexOf will return -1,
      // and scriptDirectory will correctly be replaced with an empty string.
      // If scriptDirectory contains a query (starting with ?) or a fragment (starting with #),
      // they are removed because they could contain a slash.
      if (scriptDirectory.startsWith('blob:')) {
        scriptDirectory = '';
      } else {
        scriptDirectory = scriptDirectory.substr(
          0,
          scriptDirectory.replace(/[?#].*/, '').lastIndexOf('/') + 1,
        );
      }

      if (!(typeof window == 'object' || typeof importScripts == 'function'))
        throw new Error(
          'not compiled for this environment (did you build to HTML and try to run it not on the web, or set ENVIRONMENT to something - like node - and run it someplace else - like on the web?)',
        );

      {
        // include: web_or_worker_shell_read.js
        if (ENVIRONMENT_IS_WORKER) {
          readBinary = (url) => {
            var xhr = new XMLHttpRequest();
            xhr.open('GET', url, false);
            xhr.responseType = 'arraybuffer';
            xhr.send(null);
            return new Uint8Array(/** @type{!ArrayBuffer} */ (xhr.response));
          };
        }

        readAsync = (url) => {
          assert(!isFileURI(url), 'readAsync does not work with file:// URLs');
          return fetch(url, { credentials: 'same-origin' }).then((response) => {
            if (response.ok) {
              return response.arrayBuffer();
            }
            return Promise.reject(new Error(response.status + ' : ' + response.url));
          });
        };
        // end include: web_or_worker_shell_read.js
      }
    } else {
      throw new Error('environment detection error');
    }

    var out = Module['print'] || console.log.bind(console);
    var err = Module['printErr'] || console.error.bind(console);

    // Merge back in the overrides
    Object.assign(Module, moduleOverrides);
    // Free the object hierarchy contained in the overrides, this lets the GC
    // reclaim data used.
    moduleOverrides = null;
    checkIncomingModuleAPI();

    // Emit code to handle expected values on the Module object. This applies Module.x
    // to the proper local x. This has two benefits: first, we only emit it if it is
    // expected to arrive, and second, by using a local everywhere else that can be
    // minified.

    if (Module['arguments']) arguments_ = Module['arguments'];
    legacyModuleProp('arguments', 'arguments_');

    if (Module['thisProgram']) thisProgram = Module['thisProgram'];
    legacyModuleProp('thisProgram', 'thisProgram');

    // perform assertions in shell.js after we set up out() and err(), as otherwise if an assertion fails it cannot print the message
    // Assertions on removed incoming Module JS APIs.
    assert(
      typeof Module['memoryInitializerPrefixURL'] == 'undefined',
      'Module.memoryInitializerPrefixURL option was removed, use Module.locateFile instead',
    );
    assert(
      typeof Module['pthreadMainPrefixURL'] == 'undefined',
      'Module.pthreadMainPrefixURL option was removed, use Module.locateFile instead',
    );
    assert(
      typeof Module['cdInitializerPrefixURL'] == 'undefined',
      'Module.cdInitializerPrefixURL option was removed, use Module.locateFile instead',
    );
    assert(
      typeof Module['filePackagePrefixURL'] == 'undefined',
      'Module.filePackagePrefixURL option was removed, use Module.locateFile instead',
    );
    assert(typeof Module['read'] == 'undefined', 'Module.read option was removed');
    assert(
      typeof Module['readAsync'] == 'undefined',
      'Module.readAsync option was removed (modify readAsync in JS)',
    );
    assert(
      typeof Module['readBinary'] == 'undefined',
      'Module.readBinary option was removed (modify readBinary in JS)',
    );
    assert(
      typeof Module['setWindowTitle'] == 'undefined',
      'Module.setWindowTitle option was removed (modify emscripten_set_window_title in JS)',
    );
    assert(
      typeof Module['TOTAL_MEMORY'] == 'undefined',
      'Module.TOTAL_MEMORY has been renamed Module.INITIAL_MEMORY',
    );
    legacyModuleProp('asm', 'wasmExports');
    legacyModuleProp('readAsync', 'readAsync');
    legacyModuleProp('readBinary', 'readBinary');
    legacyModuleProp('setWindowTitle', 'setWindowTitle');
    var IDBFS = 'IDBFS is no longer included by default; build with -lidbfs.js';
    var PROXYFS = 'PROXYFS is no longer included by default; build with -lproxyfs.js';
    var WORKERFS = 'WORKERFS is no longer included by default; build with -lworkerfs.js';
    var FETCHFS = 'FETCHFS is no longer included by default; build with -lfetchfs.js';
    var ICASEFS = 'ICASEFS is no longer included by default; build with -licasefs.js';
    var JSFILEFS = 'JSFILEFS is no longer included by default; build with -ljsfilefs.js';
    var OPFS = 'OPFS is no longer included by default; build with -lopfs.js';

    var NODEFS = 'NODEFS is no longer included by default; build with -lnodefs.js';

    // end include: shell.js

    // include: preamble.js
    // === Preamble library stuff ===

    // Documentation for the public APIs defined in this file must be updated in:
    //    site/source/docs/api_reference/preamble.js.rst
    // A prebuilt local version of the documentation is available at:
    //    site/build/text/docs/api_reference/preamble.js.txt
    // You can also build docs locally as HTML or other formats in site/
    // An online HTML version (which may be of a different version of Emscripten)
    //    is up at http://kripken.github.io/emscripten-site/docs/api_reference/preamble.js.html

    var wasmBinary = Module['wasmBinary'];
    legacyModuleProp('wasmBinary', 'wasmBinary');

    if (typeof WebAssembly != 'object') {
      err('no native wasm support detected');
    }

    // Wasm globals

    var wasmMemory;

    //========================================
    // Runtime essentials
    //========================================

    // whether we are quitting the application. no code should run after this.
    // set in exit() and abort()
    var ABORT = false;

    // set by exit() and abort().  Passed to 'onExit' handler.
    // NOTE: This is also used as the process return code code in shell environments
    // but only when noExitRuntime is false.
    var EXITSTATUS;

    // In STRICT mode, we only define assert() when ASSERTIONS is set.  i.e. we
    // don't define it at all in release modes.  This matches the behaviour of
    // MINIMAL_RUNTIME.
    // TODO(sbc): Make this the default even without STRICT enabled.
    /** @type {function(*, string=)} */
    function assert(condition, text) {
      if (!condition) {
        abort('Assertion failed' + (text ? ': ' + text : ''));
      }
    }

    // We used to include malloc/free by default in the past. Show a helpful error in
    // builds with assertions.

    // Memory management

    var HEAP,
      /** @type {!Int8Array} */
      HEAP8,
      /** @type {!Uint8Array} */
      HEAPU8,
      /** @type {!Int16Array} */
      HEAP16,
      /** @type {!Uint16Array} */
      HEAPU16,
      /** @type {!Int32Array} */
      HEAP32,
      /** @type {!Uint32Array} */
      HEAPU32,
      /** @type {!Float32Array} */
      HEAPF32,
      /** @type {!Float64Array} */
      HEAPF64;

    // include: runtime_shared.js
    function updateMemoryViews() {
      var b = wasmMemory.buffer;
      Module['HEAP8'] = HEAP8 = new Int8Array(b);
      Module['HEAP16'] = HEAP16 = new Int16Array(b);
      Module['HEAPU8'] = HEAPU8 = new Uint8Array(b);
      Module['HEAPU16'] = HEAPU16 = new Uint16Array(b);
      Module['HEAP32'] = HEAP32 = new Int32Array(b);
      Module['HEAPU32'] = HEAPU32 = new Uint32Array(b);
      Module['HEAPF32'] = HEAPF32 = new Float32Array(b);
      Module['HEAPF64'] = HEAPF64 = new Float64Array(b);
    }

    // end include: runtime_shared.js
    assert(
      !Module['STACK_SIZE'],
      'STACK_SIZE can no longer be set at runtime.  Use -sSTACK_SIZE at link time',
    );

    assert(
      typeof Int32Array != 'undefined' &&
        typeof Float64Array !== 'undefined' &&
        Int32Array.prototype.subarray != undefined &&
        Int32Array.prototype.set != undefined,
      'JS engine does not provide full typed array support',
    );

    // If memory is defined in wasm, the user can't provide it, or set INITIAL_MEMORY
    assert(
      !Module['wasmMemory'],
      'Use of `wasmMemory` detected.  Use -sIMPORTED_MEMORY to define wasmMemory externally',
    );
    assert(
      !Module['INITIAL_MEMORY'],
      'Detected runtime INITIAL_MEMORY setting.  Use -sIMPORTED_MEMORY to define wasmMemory dynamically',
    );

    // include: runtime_stack_check.js
    // Initializes the stack cookie. Called at the startup of main and at the startup of each thread in pthreads mode.
    function writeStackCookie() {
      var max = _emscripten_stack_get_end();
      assert((max & 3) == 0);
      // If the stack ends at address zero we write our cookies 4 bytes into the
      // stack.  This prevents interference with SAFE_HEAP and ASAN which also
      // monitor writes to address zero.
      if (max == 0) {
        max += 4;
      }
      // The stack grow downwards towards _emscripten_stack_get_end.
      // We write cookies to the final two words in the stack and detect if they are
      // ever overwritten.
      HEAPU32[max >> 2] = 0x02135467;
      HEAPU32[(max + 4) >> 2] = 0x89bacdfe;
      // Also test the global address 0 for integrity.
      HEAPU32[0 >> 2] = 1668509029;
    }

    function checkStackCookie() {
      if (ABORT) return;
      var max = _emscripten_stack_get_end();
      // See writeStackCookie().
      if (max == 0) {
        max += 4;
      }
      var cookie1 = HEAPU32[max >> 2];
      var cookie2 = HEAPU32[(max + 4) >> 2];
      if (cookie1 != 0x02135467 || cookie2 != 0x89bacdfe) {
        abort(
          `Stack overflow! Stack cookie has been overwritten at ${ptrToString(max)}, expected hex dwords 0x89BACDFE and 0x2135467, but received ${ptrToString(cookie2)} ${ptrToString(cookie1)}`,
        );
      }
      // Also test the global address 0 for integrity.
      if (HEAPU32[0 >> 2] != 0x63736d65 /* 'emsc' */) {
        abort('Runtime error: The application has corrupted its heap memory area (address zero)!');
      }
    }
    // end include: runtime_stack_check.js
    var __ATPRERUN__ = []; // functions called before the runtime is initialized
    var __ATINIT__ = []; // functions called during startup
    var __ATEXIT__ = []; // functions called during shutdown
    var __ATPOSTRUN__ = []; // functions called after the main() is called

    var runtimeInitialized = false;

    function preRun() {
      var preRuns = Module['preRun'];
      if (preRuns) {
        if (typeof preRuns == 'function') preRuns = [preRuns];
        preRuns.forEach(addOnPreRun);
      }
      callRuntimeCallbacks(__ATPRERUN__);
    }

    function initRuntime() {
      assert(!runtimeInitialized);
      runtimeInitialized = true;

      checkStackCookie();

      if (!Module['noFSInit'] && !FS.initialized) FS.init();
      FS.ignorePermissions = false;

      TTY.init();
      callRuntimeCallbacks(__ATINIT__);
    }

    function postRun() {
      checkStackCookie();

      var postRuns = Module['postRun'];
      if (postRuns) {
        if (typeof postRuns == 'function') postRuns = [postRuns];
        postRuns.forEach(addOnPostRun);
      }

      callRuntimeCallbacks(__ATPOSTRUN__);
    }

    function addOnPreRun(cb) {
      __ATPRERUN__.unshift(cb);
    }

    function addOnInit(cb) {
      __ATINIT__.unshift(cb);
    }

    function addOnExit(cb) {}

    function addOnPostRun(cb) {
      __ATPOSTRUN__.unshift(cb);
    }

    // include: runtime_math.js
    // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/imul

    // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/fround

    // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/clz32

    // https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/trunc

    assert(
      Math.imul,
      'This browser does not support Math.imul(), build with LEGACY_VM_SUPPORT or POLYFILL_OLD_MATH_FUNCTIONS to add in a polyfill',
    );
    assert(
      Math.fround,
      'This browser does not support Math.fround(), build with LEGACY_VM_SUPPORT or POLYFILL_OLD_MATH_FUNCTIONS to add in a polyfill',
    );
    assert(
      Math.clz32,
      'This browser does not support Math.clz32(), build with LEGACY_VM_SUPPORT or POLYFILL_OLD_MATH_FUNCTIONS to add in a polyfill',
    );
    assert(
      Math.trunc,
      'This browser does not support Math.trunc(), build with LEGACY_VM_SUPPORT or POLYFILL_OLD_MATH_FUNCTIONS to add in a polyfill',
    );
    // end include: runtime_math.js
    // A counter of dependencies for calling run(). If we need to
    // do asynchronous work before running, increment this and
    // decrement it. Incrementing must happen in a place like
    // Module.preRun (used by emcc to add file preloading).
    // Note that you can add dependencies in preRun, even though
    // it happens right before run - run will be postponed until
    // the dependencies are met.
    var runDependencies = 0;
    var runDependencyWatcher = null;
    var dependenciesFulfilled = null; // overridden to take different actions when all run dependencies are fulfilled
    var runDependencyTracking = {};

    function getUniqueRunDependency(id) {
      var orig = id;
      while (1) {
        if (!runDependencyTracking[id]) return id;
        id = orig + Math.random();
      }
    }

    function addRunDependency(id) {
      runDependencies++;

      Module['monitorRunDependencies']?.(runDependencies);

      if (id) {
        assert(!runDependencyTracking[id]);
        runDependencyTracking[id] = 1;
        if (runDependencyWatcher === null && typeof setInterval != 'undefined') {
          // Check for missing dependencies every few seconds
          runDependencyWatcher = setInterval(() => {
            if (ABORT) {
              clearInterval(runDependencyWatcher);
              runDependencyWatcher = null;
              return;
            }
            var shown = false;
            for (var dep in runDependencyTracking) {
              if (!shown) {
                shown = true;
                err('still waiting on run dependencies:');
              }
              err(`dependency: ${dep}`);
            }
            if (shown) {
              err('(end of list)');
            }
          }, 10000);
        }
      } else {
        err('warning: run dependency added without ID');
      }
    }

    function removeRunDependency(id) {
      runDependencies--;

      Module['monitorRunDependencies']?.(runDependencies);

      if (id) {
        assert(runDependencyTracking[id]);
        delete runDependencyTracking[id];
      } else {
        err('warning: run dependency removed without ID');
      }
      if (runDependencies == 0) {
        if (runDependencyWatcher !== null) {
          clearInterval(runDependencyWatcher);
          runDependencyWatcher = null;
        }
        if (dependenciesFulfilled) {
          var callback = dependenciesFulfilled;
          dependenciesFulfilled = null;
          callback(); // can add another dependenciesFulfilled
        }
      }
    }

    /** @param {string|number=} what */
    function abort(what) {
      Module['onAbort']?.(what);

      what = 'Aborted(' + what + ')';
      // TODO(sbc): Should we remove printing and leave it up to whoever
      // catches the exception?
      err(what);

      ABORT = true;

      // Use a wasm runtime error, because a JS error might be seen as a foreign
      // exception, which means we'd run destructors on it. We need the error to
      // simply make the program stop.
      // FIXME This approach does not work in Wasm EH because it currently does not assume
      // all RuntimeErrors are from traps; it decides whether a RuntimeError is from
      // a trap or not based on a hidden field within the object. So at the moment
      // we don't have a way of throwing a wasm trap from JS. TODO Make a JS API that
      // allows this in the wasm spec.

      // Suppress closure compiler warning here. Closure compiler's builtin extern
      // definition for WebAssembly.RuntimeError claims it takes no arguments even
      // though it can.
      // TODO(https://github.com/google/closure-compiler/pull/3913): Remove if/when upstream closure gets fixed.
      /** @suppress {checkTypes} */
      var e = new WebAssembly.RuntimeError(what);

      readyPromiseReject(e);
      // Throw the error whether or not MODULARIZE is set because abort is used
      // in code paths apart from instantiation where an exception is expected
      // to be thrown when abort is called.
      throw e;
    }

    // include: memoryprofiler.js
    // end include: memoryprofiler.js
    // include: URIUtils.js
    // Prefix of data URIs emitted by SINGLE_FILE and related options.
    var dataURIPrefix = 'data:application/octet-stream;base64,';

    /**
     * Indicates whether filename is a base64 data URI.
     * @noinline
     */
    var isDataURI = (filename) => filename.startsWith(dataURIPrefix);

    /**
     * Indicates whether filename is delivered via file protocol (as opposed to http/https)
     * @noinline
     */
    var isFileURI = (filename) => filename.startsWith('file://');
    // end include: URIUtils.js
    function createExportWrapper(name, nargs) {
      return (...args) => {
        assert(
          runtimeInitialized,
          `native function \`${name}\` called before runtime initialization`,
        );
        var f = wasmExports[name];
        assert(f, `exported native function \`${name}\` not found`);
        // Only assert for too many arguments. Too few can be valid since the missing arguments will be zero filled.
        assert(
          args.length <= nargs,
          `native function \`${name}\` called with ${args.length} args but expects ${nargs}`,
        );
        return f(...args);
      };
    }

    // include: runtime_exceptions.js
    // end include: runtime_exceptions.js
    function findWasmBinary() {
      if (Module['locateFile']) {
        var f = 'pdfium.wasm';
        if (!isDataURI(f)) {
          return locateFile(f);
        }
        return f;
      }
      if (ENVIRONMENT_IS_SHELL) return 'pdfium.wasm';
      // Use bundler-friendly `new URL(..., import.meta.url)` pattern; works in browsers too.
      return new URL('pdfium.wasm', import.meta.url).href;
    }

    var wasmBinaryFile;

    function getBinarySync(file) {
      if (file == wasmBinaryFile && wasmBinary) {
        return new Uint8Array(wasmBinary);
      }
      if (readBinary) {
        return readBinary(file);
      }
      throw 'both async and sync fetching of the wasm failed';
    }

    function getBinaryPromise(binaryFile) {
      // If we don't have the binary yet, load it asynchronously using readAsync.
      if (!wasmBinary) {
        // Fetch the binary using readAsync
        return readAsync(binaryFile).then(
          (response) => new Uint8Array(/** @type{!ArrayBuffer} */ (response)),
          // Fall back to getBinarySync if readAsync fails
          () => getBinarySync(binaryFile),
        );
      }

      // Otherwise, getBinarySync should be able to get it synchronously
      return Promise.resolve().then(() => getBinarySync(binaryFile));
    }

    function instantiateArrayBuffer(binaryFile, imports, receiver) {
      return getBinaryPromise(binaryFile)
        .then((binary) => {
          return WebAssembly.instantiate(binary, imports);
        })
        .then(receiver, (reason) => {
          err(`failed to asynchronously prepare wasm: ${reason}`);

          // Warn on some common problems.
          if (isFileURI(wasmBinaryFile)) {
            err(
              `warning: Loading from a file URI (${wasmBinaryFile}) is not supported in most browsers. See https://emscripten.org/docs/getting_started/FAQ.html#how-do-i-run-a-local-webserver-for-testing-why-does-my-program-stall-in-downloading-or-preparing`,
            );
          }
          abort(reason);
        });
    }

    function instantiateAsync(binary, binaryFile, imports, callback) {
      if (
        !binary &&
        typeof WebAssembly.instantiateStreaming == 'function' &&
        !isDataURI(binaryFile) &&
        // Avoid instantiateStreaming() on Node.js environment for now, as while
        // Node.js v18.1.0 implements it, it does not have a full fetch()
        // implementation yet.
        //
        // Reference:
        //   https://github.com/emscripten-core/emscripten/pull/16917
        !ENVIRONMENT_IS_NODE &&
        typeof fetch == 'function'
      ) {
        return fetch(binaryFile, { credentials: 'same-origin' }).then((response) => {
          // Suppress closure warning here since the upstream definition for
          // instantiateStreaming only allows Promise<Repsponse> rather than
          // an actual Response.
          // TODO(https://github.com/google/closure-compiler/pull/3913): Remove if/when upstream closure is fixed.
          /** @suppress {checkTypes} */
          var result = WebAssembly.instantiateStreaming(response, imports);

          return result.then(callback, function (reason) {
            // We expect the most common failure cause to be a bad MIME type for the binary,
            // in which case falling back to ArrayBuffer instantiation should work.
            err(`wasm streaming compile failed: ${reason}`);
            err('falling back to ArrayBuffer instantiation');
            return instantiateArrayBuffer(binaryFile, imports, callback);
          });
        });
      }
      return instantiateArrayBuffer(binaryFile, imports, callback);
    }

    function getWasmImports() {
      // prepare imports
      return {
        env: wasmImports,
        wasi_snapshot_preview1: wasmImports,
      };
    }

    // Create the wasm instance.
    // Receives the wasm imports, returns the exports.
    function createWasm() {
      var info = getWasmImports();
      // Load the wasm module and create an instance of using native support in the JS engine.
      // handle a generated wasm instance, receiving its exports and
      // performing other necessary setup
      /** @param {WebAssembly.Module=} module*/
      function receiveInstance(instance, module) {
        wasmExports = instance.exports;

        Module['wasmExports'] = wasmExports;

        wasmMemory = wasmExports['memory'];

        assert(wasmMemory, 'memory not found in wasm exports');
        updateMemoryViews();

        wasmTable = wasmExports['__indirect_function_table'];

        assert(wasmTable, 'table not found in wasm exports');

        addOnInit(wasmExports['__wasm_call_ctors']);

        removeRunDependency('wasm-instantiate');
        return wasmExports;
      }
      // wait for the pthread pool (if any)
      addRunDependency('wasm-instantiate');

      // Prefer streaming instantiation if available.
      // Async compilation can be confusing when an error on the page overwrites Module
      // (for example, if the order of elements is wrong, and the one defining Module is
      // later), so we save Module and check it later.
      var trueModule = Module;
      function receiveInstantiationResult(result) {
        // 'result' is a ResultObject object which has both the module and instance.
        // receiveInstance() will swap in the exports (to Module.asm) so they can be called
        assert(
          Module === trueModule,
          'the Module object should not be replaced during async compilation - perhaps the order of HTML elements is wrong?',
        );
        trueModule = null;
        // TODO: Due to Closure regression https://github.com/google/closure-compiler/issues/3193, the above line no longer optimizes out down to the following line.
        // When the regression is fixed, can restore the above PTHREADS-enabled path.
        receiveInstance(result['instance']);
      }

      // User shell pages can write their own Module.instantiateWasm = function(imports, successCallback) callback
      // to manually instantiate the Wasm module themselves. This allows pages to
      // run the instantiation parallel to any other async startup actions they are
      // performing.
      // Also pthreads and wasm workers initialize the wasm instance through this
      // path.
      if (Module['instantiateWasm']) {
        try {
          return Module['instantiateWasm'](info, receiveInstance);
        } catch (e) {
          err(`Module.instantiateWasm callback failed with error: ${e}`);
          // If instantiation fails, reject the module ready promise.
          readyPromiseReject(e);
        }
      }

      wasmBinaryFile ??= findWasmBinary();

      // If instantiation fails, reject the module ready promise.
      instantiateAsync(wasmBinary, wasmBinaryFile, info, receiveInstantiationResult).catch(
        readyPromiseReject,
      );
      return {}; // no exports yet; we'll fill them in later
    }

    // Globals used by JS i64 conversions (see makeSetValue)
    var tempDouble;
    var tempI64;

    // include: runtime_debug.js
    // Endianness check
    (() => {
      var h16 = new Int16Array(1);
      var h8 = new Int8Array(h16.buffer);
      h16[0] = 0x6373;
      if (h8[0] !== 0x73 || h8[1] !== 0x63)
        throw 'Runtime error: expected the system to be little-endian! (Run with -sSUPPORT_BIG_ENDIAN to bypass)';
    })();

    if (Module['ENVIRONMENT']) {
      throw new Error(
        'Module.ENVIRONMENT has been deprecated. To force the environment, use the ENVIRONMENT compile-time option (for example, -sENVIRONMENT=web or -sENVIRONMENT=node)',
      );
    }

    function legacyModuleProp(prop, newName, incoming = true) {
      if (!Object.getOwnPropertyDescriptor(Module, prop)) {
        Object.defineProperty(Module, prop, {
          configurable: true,
          get() {
            let extra = incoming
              ? ' (the initial value can be provided on Module, but after startup the value is only looked for on a local variable of that name)'
              : '';
            abort(`\`Module.${prop}\` has been replaced by \`${newName}\`` + extra);
          },
        });
      }
    }

    function ignoredModuleProp(prop) {
      if (Object.getOwnPropertyDescriptor(Module, prop)) {
        abort(
          `\`Module.${prop}\` was supplied but \`${prop}\` not included in INCOMING_MODULE_JS_API`,
        );
      }
    }

    // forcing the filesystem exports a few things by default
    function isExportedByForceFilesystem(name) {
      return (
        name === 'FS_createPath' ||
        name === 'FS_createDataFile' ||
        name === 'FS_createPreloadedFile' ||
        name === 'FS_unlink' ||
        name === 'addRunDependency' ||
        // The old FS has some functionality that WasmFS lacks.
        name === 'FS_createLazyFile' ||
        name === 'FS_createDevice' ||
        name === 'removeRunDependency'
      );
    }

    /**
     * Intercept access to a global symbol.  This enables us to give informative
     * warnings/errors when folks attempt to use symbols they did not include in
     * their build, or no symbols that no longer exist.
     */
    function hookGlobalSymbolAccess(sym, func) {
      if (typeof globalThis != 'undefined' && !Object.getOwnPropertyDescriptor(globalThis, sym)) {
        Object.defineProperty(globalThis, sym, {
          configurable: true,
          get() {
            func();
            return undefined;
          },
        });
      }
    }

    function missingGlobal(sym, msg) {
      hookGlobalSymbolAccess(sym, () => {
        warnOnce(`\`${sym}\` is not longer defined by emscripten. ${msg}`);
      });
    }

    missingGlobal('buffer', 'Please use HEAP8.buffer or wasmMemory.buffer');
    missingGlobal('asm', 'Please use wasmExports instead');

    function missingLibrarySymbol(sym) {
      hookGlobalSymbolAccess(sym, () => {
        // Can't `abort()` here because it would break code that does runtime
        // checks.  e.g. `if (typeof SDL === 'undefined')`.
        var msg = `\`${sym}\` is a library symbol and not included by default; add it to your library.js __deps or to DEFAULT_LIBRARY_FUNCS_TO_INCLUDE on the command line`;
        // DEFAULT_LIBRARY_FUNCS_TO_INCLUDE requires the name as it appears in
        // library.js, which means $name for a JS name with no prefix, or name
        // for a JS name like _name.
        var librarySymbol = sym;
        if (!librarySymbol.startsWith('_')) {
          librarySymbol = '$' + sym;
        }
        msg += ` (e.g. -sDEFAULT_LIBRARY_FUNCS_TO_INCLUDE='${librarySymbol}')`;
        if (isExportedByForceFilesystem(sym)) {
          msg +=
            '. Alternatively, forcing filesystem support (-sFORCE_FILESYSTEM) can export this for you';
        }
        warnOnce(msg);
      });

      // Any symbol that is not included from the JS library is also (by definition)
      // not exported on the Module object.
      unexportedRuntimeSymbol(sym);
    }

    function unexportedRuntimeSymbol(sym) {
      if (!Object.getOwnPropertyDescriptor(Module, sym)) {
        Object.defineProperty(Module, sym, {
          configurable: true,
          get() {
            var msg = `'${sym}' was not exported. add it to EXPORTED_RUNTIME_METHODS (see the Emscripten FAQ)`;
            if (isExportedByForceFilesystem(sym)) {
              msg +=
                '. Alternatively, forcing filesystem support (-sFORCE_FILESYSTEM) can export this for you';
            }
            abort(msg);
          },
        });
      }
    }

    // Used by XXXXX_DEBUG settings to output debug messages.
    function dbg(...args) {
      // TODO(sbc): Make this configurable somehow.  Its not always convenient for
      // logging to show up as warnings.
      console.warn(...args);
    }
    // end include: runtime_debug.js
    // === Body ===
    // end include: preamble.js

    /** @constructor */
    function ExitStatus(status) {
      this.name = 'ExitStatus';
      this.message = `Program terminated with exit(${status})`;
      this.status = status;
    }

    var callRuntimeCallbacks = (callbacks) => {
      // Pass the module as the first argument.
      callbacks.forEach((f) => f(Module));
    };

    /**
     * @param {number} ptr
     * @param {string} type
     */
    function getValue(ptr, type = 'i8') {
      if (type.endsWith('*')) type = '*';
      switch (type) {
        case 'i1':
          return HEAP8[ptr];
        case 'i8':
          return HEAP8[ptr];
        case 'i16':
          return HEAP16[ptr >> 1];
        case 'i32':
          return HEAP32[ptr >> 2];
        case 'i64':
          abort('to do getValue(i64) use WASM_BIGINT');
        case 'float':
          return HEAPF32[ptr >> 2];
        case 'double':
          return HEAPF64[ptr >> 3];
        case '*':
          return HEAPU32[ptr >> 2];
        default:
          abort(`invalid type for getValue: ${type}`);
      }
    }

    var noExitRuntime = Module['noExitRuntime'] || true;

    var ptrToString = (ptr) => {
      assert(typeof ptr === 'number');
      // With CAN_ADDRESS_2GB or MEMORY64, pointers are already unsigned.
      ptr >>>= 0;
      return '0x' + ptr.toString(16).padStart(8, '0');
    };

    /**
     * @param {number} ptr
     * @param {number} value
     * @param {string} type
     */
    function setValue(ptr, value, type = 'i8') {
      if (type.endsWith('*')) type = '*';
      switch (type) {
        case 'i1':
          HEAP8[ptr] = value;
          break;
        case 'i8':
          HEAP8[ptr] = value;
          break;
        case 'i16':
          HEAP16[ptr >> 1] = value;
          break;
        case 'i32':
          HEAP32[ptr >> 2] = value;
          break;
        case 'i64':
          abort('to do setValue(i64) use WASM_BIGINT');
        case 'float':
          HEAPF32[ptr >> 2] = value;
          break;
        case 'double':
          HEAPF64[ptr >> 3] = value;
          break;
        case '*':
          HEAPU32[ptr >> 2] = value;
          break;
        default:
          abort(`invalid type for setValue: ${type}`);
      }
    }

    var stackRestore = (val) => __emscripten_stack_restore(val);

    var stackSave = () => _emscripten_stack_get_current();

    var warnOnce = (text) => {
      warnOnce.shown ||= {};
      if (!warnOnce.shown[text]) {
        warnOnce.shown[text] = 1;
        if (ENVIRONMENT_IS_NODE) text = 'warning: ' + text;
        err(text);
      }
    };

    var UTF8Decoder = typeof TextDecoder != 'undefined' ? new TextDecoder() : undefined;

    /**
     * Given a pointer 'idx' to a null-terminated UTF8-encoded string in the given
     * array that contains uint8 values, returns a copy of that string as a
     * Javascript String object.
     * heapOrArray is either a regular array, or a JavaScript typed array view.
     * @param {number=} idx
     * @param {number=} maxBytesToRead
     * @return {string}
     */
    var UTF8ArrayToString = (heapOrArray, idx = 0, maxBytesToRead = NaN) => {
      var endIdx = idx + maxBytesToRead;
      var endPtr = idx;
      // TextDecoder needs to know the byte length in advance, it doesn't stop on
      // null terminator by itself.  Also, use the length info to avoid running tiny
      // strings through TextDecoder, since .subarray() allocates garbage.
      // (As a tiny code save trick, compare endPtr against endIdx using a negation,
      // so that undefined/NaN means Infinity)
      while (heapOrArray[endPtr] && !(endPtr >= endIdx)) ++endPtr;

      if (endPtr - idx > 16 && heapOrArray.buffer && UTF8Decoder) {
        return UTF8Decoder.decode(heapOrArray.subarray(idx, endPtr));
      }
      var str = '';
      // If building with TextDecoder, we have already computed the string length
      // above, so test loop end condition against that
      while (idx < endPtr) {
        // For UTF8 byte structure, see:
        // http://en.wikipedia.org/wiki/UTF-8#Description
        // https://www.ietf.org/rfc/rfc2279.txt
        // https://tools.ietf.org/html/rfc3629
        var u0 = heapOrArray[idx++];
        if (!(u0 & 0x80)) {
          str += String.fromCharCode(u0);
          continue;
        }
        var u1 = heapOrArray[idx++] & 63;
        if ((u0 & 0xe0) == 0xc0) {
          str += String.fromCharCode(((u0 & 31) << 6) | u1);
          continue;
        }
        var u2 = heapOrArray[idx++] & 63;
        if ((u0 & 0xf0) == 0xe0) {
          u0 = ((u0 & 15) << 12) | (u1 << 6) | u2;
        } else {
          if ((u0 & 0xf8) != 0xf0)
            warnOnce(
              'Invalid UTF-8 leading byte ' +
                ptrToString(u0) +
                ' encountered when deserializing a UTF-8 string in wasm memory to a JS string!',
            );
          u0 = ((u0 & 7) << 18) | (u1 << 12) | (u2 << 6) | (heapOrArray[idx++] & 63);
        }

        if (u0 < 0x10000) {
          str += String.fromCharCode(u0);
        } else {
          var ch = u0 - 0x10000;
          str += String.fromCharCode(0xd800 | (ch >> 10), 0xdc00 | (ch & 0x3ff));
        }
      }
      return str;
    };

    /**
     * Given a pointer 'ptr' to a null-terminated UTF8-encoded string in the
     * emscripten HEAP, returns a copy of that string as a Javascript String object.
     *
     * @param {number} ptr
     * @param {number=} maxBytesToRead - An optional length that specifies the
     *   maximum number of bytes to read. You can omit this parameter to scan the
     *   string until the first 0 byte. If maxBytesToRead is passed, and the string
     *   at [ptr, ptr+maxBytesToReadr[ contains a null byte in the middle, then the
     *   string will cut short at that byte index (i.e. maxBytesToRead will not
     *   produce a string of exact length [ptr, ptr+maxBytesToRead[) N.B. mixing
     *   frequent uses of UTF8ToString() with and without maxBytesToRead may throw
     *   JS JIT optimizations off, so it is worth to consider consistently using one
     * @return {string}
     */
    var UTF8ToString = (ptr, maxBytesToRead) => {
      assert(typeof ptr == 'number', `UTF8ToString expects a number (got ${typeof ptr})`);
      return ptr ? UTF8ArrayToString(HEAPU8, ptr, maxBytesToRead) : '';
    };
    var ___assert_fail = (condition, filename, line, func) => {
      abort(
        `Assertion failed: ${UTF8ToString(condition)}, at: ` +
          [
            filename ? UTF8ToString(filename) : 'unknown filename',
            line,
            func ? UTF8ToString(func) : 'unknown function',
          ],
      );
    };

    /** @suppress {duplicate } */
    function syscallGetVarargI() {
      assert(SYSCALLS.varargs != undefined);
      // the `+` prepended here is necessary to convince the JSCompiler that varargs is indeed a number.
      var ret = HEAP32[+SYSCALLS.varargs >> 2];
      SYSCALLS.varargs += 4;
      return ret;
    }
    var syscallGetVarargP = syscallGetVarargI;

    var PATH = {
      isAbs: (path) => path.charAt(0) === '/',
      splitPath: (filename) => {
        var splitPathRe = /^(\/?|)([\s\S]*?)((?:\.{1,2}|[^\/]+?|)(\.[^.\/]*|))(?:[\/]*)$/;
        return splitPathRe.exec(filename).slice(1);
      },
      normalizeArray: (parts, allowAboveRoot) => {
        // if the path tries to go above the root, `up` ends up > 0
        var up = 0;
        for (var i = parts.length - 1; i >= 0; i--) {
          var last = parts[i];
          if (last === '.') {
            parts.splice(i, 1);
          } else if (last === '..') {
            parts.splice(i, 1);
            up++;
          } else if (up) {
            parts.splice(i, 1);
            up--;
          }
        }
        // if the path is allowed to go above the root, restore leading ..s
        if (allowAboveRoot) {
          for (; up; up--) {
            parts.unshift('..');
          }
        }
        return parts;
      },
      normalize: (path) => {
        var isAbsolute = PATH.isAbs(path),
          trailingSlash = path.substr(-1) === '/';
        // Normalize the path
        path = PATH.normalizeArray(
          path.split('/').filter((p) => !!p),
          !isAbsolute,
        ).join('/');
        if (!path && !isAbsolute) {
          path = '.';
        }
        if (path && trailingSlash) {
          path += '/';
        }
        return (isAbsolute ? '/' : '') + path;
      },
      dirname: (path) => {
        var result = PATH.splitPath(path),
          root = result[0],
          dir = result[1];
        if (!root && !dir) {
          // No dirname whatsoever
          return '.';
        }
        if (dir) {
          // It has a dirname, strip trailing slash
          dir = dir.substr(0, dir.length - 1);
        }
        return root + dir;
      },
      basename: (path) => {
        // EMSCRIPTEN return '/'' for '/', not an empty string
        if (path === '/') return '/';
        path = PATH.normalize(path);
        path = path.replace(/\/$/, '');
        var lastSlash = path.lastIndexOf('/');
        if (lastSlash === -1) return path;
        return path.substr(lastSlash + 1);
      },
      join: (...paths) => PATH.normalize(paths.join('/')),
      join2: (l, r) => PATH.normalize(l + '/' + r),
    };

    var initRandomFill = () => {
      if (typeof crypto == 'object' && typeof crypto['getRandomValues'] == 'function') {
        // for modern web browsers
        return (view) => crypto.getRandomValues(view);
      } else if (ENVIRONMENT_IS_NODE) {
        // for nodejs with or without crypto support included
        try {
          var crypto_module = require('crypto');
          var randomFillSync = crypto_module['randomFillSync'];
          if (randomFillSync) {
            // nodejs with LTS crypto support
            return (view) => crypto_module['randomFillSync'](view);
          }
          // very old nodejs with the original crypto API
          var randomBytes = crypto_module['randomBytes'];
          return (view) => (
            view.set(randomBytes(view.byteLength)),
            // Return the original view to match modern native implementations.
            view
          );
        } catch (e) {
          // nodejs doesn't have crypto support
        }
      }
      // we couldn't find a proper implementation, as Math.random() is not suitable for /dev/random, see emscripten-core/emscripten/pull/7096
      abort(
        'no cryptographic support found for randomDevice. consider polyfilling it if you want to use something insecure like Math.random(), e.g. put this in a --pre-js: var crypto = { getRandomValues: (array) => { for (var i = 0; i < array.length; i++) array[i] = (Math.random()*256)|0 } };',
      );
    };
    var randomFill = (view) => {
      // Lazily init on the first invocation.
      return (randomFill = initRandomFill())(view);
    };

    var PATH_FS = {
      resolve: (...args) => {
        var resolvedPath = '',
          resolvedAbsolute = false;
        for (var i = args.length - 1; i >= -1 && !resolvedAbsolute; i--) {
          var path = i >= 0 ? args[i] : FS.cwd();
          // Skip empty and invalid entries
          if (typeof path != 'string') {
            throw new TypeError('Arguments to path.resolve must be strings');
          } else if (!path) {
            return ''; // an invalid portion invalidates the whole thing
          }
          resolvedPath = path + '/' + resolvedPath;
          resolvedAbsolute = PATH.isAbs(path);
        }
        // At this point the path should be resolved to a full absolute path, but
        // handle relative paths to be safe (might happen when process.cwd() fails)
        resolvedPath = PATH.normalizeArray(
          resolvedPath.split('/').filter((p) => !!p),
          !resolvedAbsolute,
        ).join('/');
        return (resolvedAbsolute ? '/' : '') + resolvedPath || '.';
      },
      relative: (from, to) => {
        from = PATH_FS.resolve(from).substr(1);
        to = PATH_FS.resolve(to).substr(1);
        function trim(arr) {
          var start = 0;
          for (; start < arr.length; start++) {
            if (arr[start] !== '') break;
          }
          var end = arr.length - 1;
          for (; end >= 0; end--) {
            if (arr[end] !== '') break;
          }
          if (start > end) return [];
          return arr.slice(start, end - start + 1);
        }
        var fromParts = trim(from.split('/'));
        var toParts = trim(to.split('/'));
        var length = Math.min(fromParts.length, toParts.length);
        var samePartsLength = length;
        for (var i = 0; i < length; i++) {
          if (fromParts[i] !== toParts[i]) {
            samePartsLength = i;
            break;
          }
        }
        var outputParts = [];
        for (var i = samePartsLength; i < fromParts.length; i++) {
          outputParts.push('..');
        }
        outputParts = outputParts.concat(toParts.slice(samePartsLength));
        return outputParts.join('/');
      },
    };

    var FS_stdin_getChar_buffer = [];

    var lengthBytesUTF8 = (str) => {
      var len = 0;
      for (var i = 0; i < str.length; ++i) {
        // Gotcha: charCodeAt returns a 16-bit word that is a UTF-16 encoded code
        // unit, not a Unicode code point of the character! So decode
        // UTF16->UTF32->UTF8.
        // See http://unicode.org/faq/utf_bom.html#utf16-3
        var c = str.charCodeAt(i); // possibly a lead surrogate
        if (c <= 0x7f) {
          len++;
        } else if (c <= 0x7ff) {
          len += 2;
        } else if (c >= 0xd800 && c <= 0xdfff) {
          len += 4;
          ++i;
        } else {
          len += 3;
        }
      }
      return len;
    };

    var stringToUTF8Array = (str, heap, outIdx, maxBytesToWrite) => {
      assert(typeof str === 'string', `stringToUTF8Array expects a string (got ${typeof str})`);
      // Parameter maxBytesToWrite is not optional. Negative values, 0, null,
      // undefined and false each don't write out any bytes.
      if (!(maxBytesToWrite > 0)) return 0;

      var startIdx = outIdx;
      var endIdx = outIdx + maxBytesToWrite - 1; // -1 for string null terminator.
      for (var i = 0; i < str.length; ++i) {
        // Gotcha: charCodeAt returns a 16-bit word that is a UTF-16 encoded code
        // unit, not a Unicode code point of the character! So decode
        // UTF16->UTF32->UTF8.
        // See http://unicode.org/faq/utf_bom.html#utf16-3
        // For UTF8 byte structure, see http://en.wikipedia.org/wiki/UTF-8#Description
        // and https://www.ietf.org/rfc/rfc2279.txt
        // and https://tools.ietf.org/html/rfc3629
        var u = str.charCodeAt(i); // possibly a lead surrogate
        if (u >= 0xd800 && u <= 0xdfff) {
          var u1 = str.charCodeAt(++i);
          u = (0x10000 + ((u & 0x3ff) << 10)) | (u1 & 0x3ff);
        }
        if (u <= 0x7f) {
          if (outIdx >= endIdx) break;
          heap[outIdx++] = u;
        } else if (u <= 0x7ff) {
          if (outIdx + 1 >= endIdx) break;
          heap[outIdx++] = 0xc0 | (u >> 6);
          heap[outIdx++] = 0x80 | (u & 63);
        } else if (u <= 0xffff) {
          if (outIdx + 2 >= endIdx) break;
          heap[outIdx++] = 0xe0 | (u >> 12);
          heap[outIdx++] = 0x80 | ((u >> 6) & 63);
          heap[outIdx++] = 0x80 | (u & 63);
        } else {
          if (outIdx + 3 >= endIdx) break;
          if (u > 0x10ffff)
            warnOnce(
              'Invalid Unicode code point ' +
                ptrToString(u) +
                ' encountered when serializing a JS string to a UTF-8 string in wasm memory! (Valid unicode code points should be in range 0-0x10FFFF).',
            );
          heap[outIdx++] = 0xf0 | (u >> 18);
          heap[outIdx++] = 0x80 | ((u >> 12) & 63);
          heap[outIdx++] = 0x80 | ((u >> 6) & 63);
          heap[outIdx++] = 0x80 | (u & 63);
        }
      }
      // Null-terminate the pointer to the buffer.
      heap[outIdx] = 0;
      return outIdx - startIdx;
    };
    /** @type {function(string, boolean=, number=)} */
    function intArrayFromString(stringy, dontAddNull, length) {
      var len = length > 0 ? length : lengthBytesUTF8(stringy) + 1;
      var u8array = new Array(len);
      var numBytesWritten = stringToUTF8Array(stringy, u8array, 0, u8array.length);
      if (dontAddNull) u8array.length = numBytesWritten;
      return u8array;
    }
    var FS_stdin_getChar = () => {
      if (!FS_stdin_getChar_buffer.length) {
        var result = null;
        if (ENVIRONMENT_IS_NODE) {
          // we will read data by chunks of BUFSIZE
          var BUFSIZE = 256;
          var buf = Buffer.alloc(BUFSIZE);
          var bytesRead = 0;

          // For some reason we must suppress a closure warning here, even though
          // fd definitely exists on process.stdin, and is even the proper way to
          // get the fd of stdin,
          // https://github.com/nodejs/help/issues/2136#issuecomment-523649904
          // This started to happen after moving this logic out of library_tty.js,
          // so it is related to the surrounding code in some unclear manner.
          /** @suppress {missingProperties} */
          var fd = process.stdin.fd;

          try {
            bytesRead = fs.readSync(fd, buf, 0, BUFSIZE);
          } catch (e) {
            // Cross-platform differences: on Windows, reading EOF throws an
            // exception, but on other OSes, reading EOF returns 0. Uniformize
            // behavior by treating the EOF exception to return 0.
            if (e.toString().includes('EOF')) bytesRead = 0;
            else throw e;
          }

          if (bytesRead > 0) {
            result = buf.slice(0, bytesRead).toString('utf-8');
          }
        } else if (typeof window != 'undefined' && typeof window.prompt == 'function') {
          // Browser.
          result = window.prompt('Input: '); // returns null on cancel
          if (result !== null) {
            result += '\n';
          }
        } else if (typeof readline == 'function') {
          // Command line.
          result = readline();
          if (result) {
            result += '\n';
          }
        } else {
        }
        if (!result) {
          return null;
        }
        FS_stdin_getChar_buffer = intArrayFromString(result, true);
      }
      return FS_stdin_getChar_buffer.shift();
    };
    var TTY = {
      ttys: [],
      init() {
        // https://github.com/emscripten-core/emscripten/pull/1555
        // if (ENVIRONMENT_IS_NODE) {
        //   // currently, FS.init does not distinguish if process.stdin is a file or TTY
        //   // device, it always assumes it's a TTY device. because of this, we're forcing
        //   // process.stdin to UTF8 encoding to at least make stdin reading compatible
        //   // with text files until FS.init can be refactored.
        //   process.stdin.setEncoding('utf8');
        // }
      },
      shutdown() {
        // https://github.com/emscripten-core/emscripten/pull/1555
        // if (ENVIRONMENT_IS_NODE) {
        //   // inolen: any idea as to why node -e 'process.stdin.read()' wouldn't exit immediately (with process.stdin being a tty)?
        //   // isaacs: because now it's reading from the stream, you've expressed interest in it, so that read() kicks off a _read() which creates a ReadReq operation
        //   // inolen: I thought read() in that case was a synchronous operation that just grabbed some amount of buffered data if it exists?
        //   // isaacs: it is. but it also triggers a _read() call, which calls readStart() on the handle
        //   // isaacs: do process.stdin.pause() and i'd think it'd probably close the pending call
        //   process.stdin.pause();
        // }
      },
      register(dev, ops) {
        TTY.ttys[dev] = { input: [], output: [], ops: ops };
        FS.registerDevice(dev, TTY.stream_ops);
      },
      stream_ops: {
        open(stream) {
          var tty = TTY.ttys[stream.node.rdev];
          if (!tty) {
            throw new FS.ErrnoError(43);
          }
          stream.tty = tty;
          stream.seekable = false;
        },
        close(stream) {
          // flush any pending line data
          stream.tty.ops.fsync(stream.tty);
        },
        fsync(stream) {
          stream.tty.ops.fsync(stream.tty);
        },
        read(stream, buffer, offset, length, pos /* ignored */) {
          if (!stream.tty || !stream.tty.ops.get_char) {
            throw new FS.ErrnoError(60);
          }
          var bytesRead = 0;
          for (var i = 0; i < length; i++) {
            var result;
            try {
              result = stream.tty.ops.get_char(stream.tty);
            } catch (e) {
              throw new FS.ErrnoError(29);
            }
            if (result === undefined && bytesRead === 0) {
              throw new FS.ErrnoError(6);
            }
            if (result === null || result === undefined) break;
            bytesRead++;
            buffer[offset + i] = result;
          }
          if (bytesRead) {
            stream.node.timestamp = Date.now();
          }
          return bytesRead;
        },
        write(stream, buffer, offset, length, pos) {
          if (!stream.tty || !stream.tty.ops.put_char) {
            throw new FS.ErrnoError(60);
          }
          try {
            for (var i = 0; i < length; i++) {
              stream.tty.ops.put_char(stream.tty, buffer[offset + i]);
            }
          } catch (e) {
            throw new FS.ErrnoError(29);
          }
          if (length) {
            stream.node.timestamp = Date.now();
          }
          return i;
        },
      },
      default_tty_ops: {
        get_char(tty) {
          return FS_stdin_getChar();
        },
        put_char(tty, val) {
          if (val === null || val === 10) {
            out(UTF8ArrayToString(tty.output));
            tty.output = [];
          } else {
            if (val != 0) tty.output.push(val); // val == 0 would cut text output off in the middle.
          }
        },
        fsync(tty) {
          if (tty.output && tty.output.length > 0) {
            out(UTF8ArrayToString(tty.output));
            tty.output = [];
          }
        },
        ioctl_tcgets(tty) {
          // typical setting
          return {
            c_iflag: 25856,
            c_oflag: 5,
            c_cflag: 191,
            c_lflag: 35387,
            c_cc: [
              0x03, 0x1c, 0x7f, 0x15, 0x04, 0x00, 0x01, 0x00, 0x11, 0x13, 0x1a, 0x00, 0x12, 0x0f,
              0x17, 0x16, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
              0x00, 0x00, 0x00, 0x00,
            ],
          };
        },
        ioctl_tcsets(tty, optional_actions, data) {
          // currently just ignore
          return 0;
        },
        ioctl_tiocgwinsz(tty) {
          return [24, 80];
        },
      },
      default_tty1_ops: {
        put_char(tty, val) {
          if (val === null || val === 10) {
            err(UTF8ArrayToString(tty.output));
            tty.output = [];
          } else {
            if (val != 0) tty.output.push(val);
          }
        },
        fsync(tty) {
          if (tty.output && tty.output.length > 0) {
            err(UTF8ArrayToString(tty.output));
            tty.output = [];
          }
        },
      },
    };

    var zeroMemory = (address, size) => {
      HEAPU8.fill(0, address, address + size);
    };

    var alignMemory = (size, alignment) => {
      assert(alignment, 'alignment argument is required');
      return Math.ceil(size / alignment) * alignment;
    };
    var mmapAlloc = (size) => {
      size = alignMemory(size, 65536);
      var ptr = _emscripten_builtin_memalign(65536, size);
      if (ptr) zeroMemory(ptr, size);
      return ptr;
    };
    var MEMFS = {
      ops_table: null,
      mount(mount) {
        return MEMFS.createNode(null, '/', 16384 | 511 /* 0777 */, 0);
      },
      createNode(parent, name, mode, dev) {
        if (FS.isBlkdev(mode) || FS.isFIFO(mode)) {
          // no supported
          throw new FS.ErrnoError(63);
        }
        MEMFS.ops_table ||= {
          dir: {
            node: {
              getattr: MEMFS.node_ops.getattr,
              setattr: MEMFS.node_ops.setattr,
              lookup: MEMFS.node_ops.lookup,
              mknod: MEMFS.node_ops.mknod,
              rename: MEMFS.node_ops.rename,
              unlink: MEMFS.node_ops.unlink,
              rmdir: MEMFS.node_ops.rmdir,
              readdir: MEMFS.node_ops.readdir,
              symlink: MEMFS.node_ops.symlink,
            },
            stream: {
              llseek: MEMFS.stream_ops.llseek,
            },
          },
          file: {
            node: {
              getattr: MEMFS.node_ops.getattr,
              setattr: MEMFS.node_ops.setattr,
            },
            stream: {
              llseek: MEMFS.stream_ops.llseek,
              read: MEMFS.stream_ops.read,
              write: MEMFS.stream_ops.write,
              allocate: MEMFS.stream_ops.allocate,
              mmap: MEMFS.stream_ops.mmap,
              msync: MEMFS.stream_ops.msync,
            },
          },
          link: {
            node: {
              getattr: MEMFS.node_ops.getattr,
              setattr: MEMFS.node_ops.setattr,
              readlink: MEMFS.node_ops.readlink,
            },
            stream: {},
          },
          chrdev: {
            node: {
              getattr: MEMFS.node_ops.getattr,
              setattr: MEMFS.node_ops.setattr,
            },
            stream: FS.chrdev_stream_ops,
          },
        };
        var node = FS.createNode(parent, name, mode, dev);
        if (FS.isDir(node.mode)) {
          node.node_ops = MEMFS.ops_table.dir.node;
          node.stream_ops = MEMFS.ops_table.dir.stream;
          node.contents = {};
        } else if (FS.isFile(node.mode)) {
          node.node_ops = MEMFS.ops_table.file.node;
          node.stream_ops = MEMFS.ops_table.file.stream;
          node.usedBytes = 0; // The actual number of bytes used in the typed array, as opposed to contents.length which gives the whole capacity.
          // When the byte data of the file is populated, this will point to either a typed array, or a normal JS array. Typed arrays are preferred
          // for performance, and used by default. However, typed arrays are not resizable like normal JS arrays are, so there is a small disk size
          // penalty involved for appending file writes that continuously grow a file similar to std::vector capacity vs used -scheme.
          node.contents = null;
        } else if (FS.isLink(node.mode)) {
          node.node_ops = MEMFS.ops_table.link.node;
          node.stream_ops = MEMFS.ops_table.link.stream;
        } else if (FS.isChrdev(node.mode)) {
          node.node_ops = MEMFS.ops_table.chrdev.node;
          node.stream_ops = MEMFS.ops_table.chrdev.stream;
        }
        node.timestamp = Date.now();
        // add the new node to the parent
        if (parent) {
          parent.contents[name] = node;
          parent.timestamp = node.timestamp;
        }
        return node;
      },
      getFileDataAsTypedArray(node) {
        if (!node.contents) return new Uint8Array(0);
        if (node.contents.subarray) return node.contents.subarray(0, node.usedBytes); // Make sure to not return excess unused bytes.
        return new Uint8Array(node.contents);
      },
      expandFileStorage(node, newCapacity) {
        var prevCapacity = node.contents ? node.contents.length : 0;
        if (prevCapacity >= newCapacity) return; // No need to expand, the storage was already large enough.
        // Don't expand strictly to the given requested limit if it's only a very small increase, but instead geometrically grow capacity.
        // For small filesizes (<1MB), perform size*2 geometric increase, but for large sizes, do a much more conservative size*1.125 increase to
        // avoid overshooting the allocation cap by a very large margin.
        var CAPACITY_DOUBLING_MAX = 1024 * 1024;
        newCapacity = Math.max(
          newCapacity,
          (prevCapacity * (prevCapacity < CAPACITY_DOUBLING_MAX ? 2.0 : 1.125)) >>> 0,
        );
        if (prevCapacity != 0) newCapacity = Math.max(newCapacity, 256); // At minimum allocate 256b for each file when expanding.
        var oldContents = node.contents;
        node.contents = new Uint8Array(newCapacity); // Allocate new storage.
        if (node.usedBytes > 0) node.contents.set(oldContents.subarray(0, node.usedBytes), 0); // Copy old data over to the new storage.
      },
      resizeFileStorage(node, newSize) {
        if (node.usedBytes == newSize) return;
        if (newSize == 0) {
          node.contents = null; // Fully decommit when requesting a resize to zero.
          node.usedBytes = 0;
        } else {
          var oldContents = node.contents;
          node.contents = new Uint8Array(newSize); // Allocate new storage.
          if (oldContents) {
            node.contents.set(oldContents.subarray(0, Math.min(newSize, node.usedBytes))); // Copy old data over to the new storage.
          }
          node.usedBytes = newSize;
        }
      },
      node_ops: {
        getattr(node) {
          var attr = {};
          // device numbers reuse inode numbers.
          attr.dev = FS.isChrdev(node.mode) ? node.id : 1;
          attr.ino = node.id;
          attr.mode = node.mode;
          attr.nlink = 1;
          attr.uid = 0;
          attr.gid = 0;
          attr.rdev = node.rdev;
          if (FS.isDir(node.mode)) {
            attr.size = 4096;
          } else if (FS.isFile(node.mode)) {
            attr.size = node.usedBytes;
          } else if (FS.isLink(node.mode)) {
            attr.size = node.link.length;
          } else {
            attr.size = 0;
          }
          attr.atime = new Date(node.timestamp);
          attr.mtime = new Date(node.timestamp);
          attr.ctime = new Date(node.timestamp);
          // NOTE: In our implementation, st_blocks = Math.ceil(st_size/st_blksize),
          //       but this is not required by the standard.
          attr.blksize = 4096;
          attr.blocks = Math.ceil(attr.size / attr.blksize);
          return attr;
        },
        setattr(node, attr) {
          if (attr.mode !== undefined) {
            node.mode = attr.mode;
          }
          if (attr.timestamp !== undefined) {
            node.timestamp = attr.timestamp;
          }
          if (attr.size !== undefined) {
            MEMFS.resizeFileStorage(node, attr.size);
          }
        },
        lookup(parent, name) {
          throw FS.genericErrors[44];
        },
        mknod(parent, name, mode, dev) {
          return MEMFS.createNode(parent, name, mode, dev);
        },
        rename(old_node, new_dir, new_name) {
          // if we're overwriting a directory at new_name, make sure it's empty.
          if (FS.isDir(old_node.mode)) {
            var new_node;
            try {
              new_node = FS.lookupNode(new_dir, new_name);
            } catch (e) {}
            if (new_node) {
              for (var i in new_node.contents) {
                throw new FS.ErrnoError(55);
              }
            }
          }
          // do the internal rewiring
          delete old_node.parent.contents[old_node.name];
          old_node.parent.timestamp = Date.now();
          old_node.name = new_name;
          new_dir.contents[new_name] = old_node;
          new_dir.timestamp = old_node.parent.timestamp;
        },
        unlink(parent, name) {
          delete parent.contents[name];
          parent.timestamp = Date.now();
        },
        rmdir(parent, name) {
          var node = FS.lookupNode(parent, name);
          for (var i in node.contents) {
            throw new FS.ErrnoError(55);
          }
          delete parent.contents[name];
          parent.timestamp = Date.now();
        },
        readdir(node) {
          var entries = ['.', '..'];
          for (var key of Object.keys(node.contents)) {
            entries.push(key);
          }
          return entries;
        },
        symlink(parent, newname, oldpath) {
          var node = MEMFS.createNode(parent, newname, 511 /* 0777 */ | 40960, 0);
          node.link = oldpath;
          return node;
        },
        readlink(node) {
          if (!FS.isLink(node.mode)) {
            throw new FS.ErrnoError(28);
          }
          return node.link;
        },
      },
      stream_ops: {
        read(stream, buffer, offset, length, position) {
          var contents = stream.node.contents;
          if (position >= stream.node.usedBytes) return 0;
          var size = Math.min(stream.node.usedBytes - position, length);
          assert(size >= 0);
          if (size > 8 && contents.subarray) {
            // non-trivial, and typed array
            buffer.set(contents.subarray(position, position + size), offset);
          } else {
            for (var i = 0; i < size; i++) buffer[offset + i] = contents[position + i];
          }
          return size;
        },
        write(stream, buffer, offset, length, position, canOwn) {
          // The data buffer should be a typed array view
          assert(!(buffer instanceof ArrayBuffer));
          // If the buffer is located in main memory (HEAP), and if
          // memory can grow, we can't hold on to references of the
          // memory buffer, as they may get invalidated. That means we
          // need to do copy its contents.
          if (buffer.buffer === HEAP8.buffer) {
            canOwn = false;
          }

          if (!length) return 0;
          var node = stream.node;
          node.timestamp = Date.now();

          if (buffer.subarray && (!node.contents || node.contents.subarray)) {
            // This write is from a typed array to a typed array?
            if (canOwn) {
              assert(position === 0, 'canOwn must imply no weird position inside the file');
              node.contents = buffer.subarray(offset, offset + length);
              node.usedBytes = length;
              return length;
            } else if (node.usedBytes === 0 && position === 0) {
              // If this is a simple first write to an empty file, do a fast set since we don't need to care about old data.
              node.contents = buffer.slice(offset, offset + length);
              node.usedBytes = length;
              return length;
            } else if (position + length <= node.usedBytes) {
              // Writing to an already allocated and used subrange of the file?
              node.contents.set(buffer.subarray(offset, offset + length), position);
              return length;
            }
          }

          // Appending to an existing file and we need to reallocate, or source data did not come as a typed array.
          MEMFS.expandFileStorage(node, position + length);
          if (node.contents.subarray && buffer.subarray) {
            // Use typed array write which is available.
            node.contents.set(buffer.subarray(offset, offset + length), position);
          } else {
            for (var i = 0; i < length; i++) {
              node.contents[position + i] = buffer[offset + i]; // Or fall back to manual write if not.
            }
          }
          node.usedBytes = Math.max(node.usedBytes, position + length);
          return length;
        },
        llseek(stream, offset, whence) {
          var position = offset;
          if (whence === 1) {
            position += stream.position;
          } else if (whence === 2) {
            if (FS.isFile(stream.node.mode)) {
              position += stream.node.usedBytes;
            }
          }
          if (position < 0) {
            throw new FS.ErrnoError(28);
          }
          return position;
        },
        allocate(stream, offset, length) {
          MEMFS.expandFileStorage(stream.node, offset + length);
          stream.node.usedBytes = Math.max(stream.node.usedBytes, offset + length);
        },
        mmap(stream, length, position, prot, flags) {
          if (!FS.isFile(stream.node.mode)) {
            throw new FS.ErrnoError(43);
          }
          var ptr;
          var allocated;
          var contents = stream.node.contents;
          // Only make a new copy when MAP_PRIVATE is specified.
          if (!(flags & 2) && contents && contents.buffer === HEAP8.buffer) {
            // We can't emulate MAP_SHARED when the file is not backed by the
            // buffer we're mapping to (e.g. the HEAP buffer).
            allocated = false;
            ptr = contents.byteOffset;
          } else {
            allocated = true;
            ptr = mmapAlloc(length);
            if (!ptr) {
              throw new FS.ErrnoError(48);
            }
            if (contents) {
              // Try to avoid unnecessary slices.
              if (position > 0 || position + length < contents.length) {
                if (contents.subarray) {
                  contents = contents.subarray(position, position + length);
                } else {
                  contents = Array.prototype.slice.call(contents, position, position + length);
                }
              }
              HEAP8.set(contents, ptr);
            }
          }
          return { ptr, allocated };
        },
        msync(stream, buffer, offset, length, mmapFlags) {
          MEMFS.stream_ops.write(stream, buffer, 0, length, offset, false);
          // should we check if bytesWritten and length are the same?
          return 0;
        },
      },
    };

    /** @param {boolean=} noRunDep */
    var asyncLoad = (url, onload, onerror, noRunDep) => {
      var dep = !noRunDep ? getUniqueRunDependency(`al ${url}`) : '';
      readAsync(url).then(
        (arrayBuffer) => {
          assert(arrayBuffer, `Loading data file "${url}" failed (no arrayBuffer).`);
          onload(new Uint8Array(arrayBuffer));
          if (dep) removeRunDependency(dep);
        },
        (err) => {
          if (onerror) {
            onerror();
          } else {
            throw `Loading data file "${url}" failed.`;
          }
        },
      );
      if (dep) addRunDependency(dep);
    };

    var FS_createDataFile = (parent, name, fileData, canRead, canWrite, canOwn) => {
      FS.createDataFile(parent, name, fileData, canRead, canWrite, canOwn);
    };

    var preloadPlugins = Module['preloadPlugins'] || [];
    var FS_handledByPreloadPlugin = (byteArray, fullname, finish, onerror) => {
      // Ensure plugins are ready.
      if (typeof Browser != 'undefined') Browser.init();

      var handled = false;
      preloadPlugins.forEach((plugin) => {
        if (handled) return;
        if (plugin['canHandle'](fullname)) {
          plugin['handle'](byteArray, fullname, finish, onerror);
          handled = true;
        }
      });
      return handled;
    };
    var FS_createPreloadedFile = (
      parent,
      name,
      url,
      canRead,
      canWrite,
      onload,
      onerror,
      dontCreateFile,
      canOwn,
      preFinish,
    ) => {
      // TODO we should allow people to just pass in a complete filename instead
      // of parent and name being that we just join them anyways
      var fullname = name ? PATH_FS.resolve(PATH.join2(parent, name)) : parent;
      var dep = getUniqueRunDependency(`cp ${fullname}`); // might have several active requests for the same fullname
      function processData(byteArray) {
        function finish(byteArray) {
          preFinish?.();
          if (!dontCreateFile) {
            FS_createDataFile(parent, name, byteArray, canRead, canWrite, canOwn);
          }
          onload?.();
          removeRunDependency(dep);
        }
        if (
          FS_handledByPreloadPlugin(byteArray, fullname, finish, () => {
            onerror?.();
            removeRunDependency(dep);
          })
        ) {
          return;
        }
        finish(byteArray);
      }
      addRunDependency(dep);
      if (typeof url == 'string') {
        asyncLoad(url, processData, onerror);
      } else {
        processData(url);
      }
    };

    var FS_modeStringToFlags = (str) => {
      var flagModes = {
        r: 0,
        'r+': 2,
        w: 512 | 64 | 1,
        'w+': 512 | 64 | 2,
        a: 1024 | 64 | 1,
        'a+': 1024 | 64 | 2,
      };
      var flags = flagModes[str];
      if (typeof flags == 'undefined') {
        throw new Error(`Unknown file open mode: ${str}`);
      }
      return flags;
    };

    var FS_getMode = (canRead, canWrite) => {
      var mode = 0;
      if (canRead) mode |= 292 | 73;
      if (canWrite) mode |= 146;
      return mode;
    };

    var strError = (errno) => {
      return UTF8ToString(_strerror(errno));
    };

    var ERRNO_CODES = {
      EPERM: 63,
      ENOENT: 44,
      ESRCH: 71,
      EINTR: 27,
      EIO: 29,
      ENXIO: 60,
      E2BIG: 1,
      ENOEXEC: 45,
      EBADF: 8,
      ECHILD: 12,
      EAGAIN: 6,
      EWOULDBLOCK: 6,
      ENOMEM: 48,
      EACCES: 2,
      EFAULT: 21,
      ENOTBLK: 105,
      EBUSY: 10,
      EEXIST: 20,
      EXDEV: 75,
      ENODEV: 43,
      ENOTDIR: 54,
      EISDIR: 31,
      EINVAL: 28,
      ENFILE: 41,
      EMFILE: 33,
      ENOTTY: 59,
      ETXTBSY: 74,
      EFBIG: 22,
      ENOSPC: 51,
      ESPIPE: 70,
      EROFS: 69,
      EMLINK: 34,
      EPIPE: 64,
      EDOM: 18,
      ERANGE: 68,
      ENOMSG: 49,
      EIDRM: 24,
      ECHRNG: 106,
      EL2NSYNC: 156,
      EL3HLT: 107,
      EL3RST: 108,
      ELNRNG: 109,
      EUNATCH: 110,
      ENOCSI: 111,
      EL2HLT: 112,
      EDEADLK: 16,
      ENOLCK: 46,
      EBADE: 113,
      EBADR: 114,
      EXFULL: 115,
      ENOANO: 104,
      EBADRQC: 103,
      EBADSLT: 102,
      EDEADLOCK: 16,
      EBFONT: 101,
      ENOSTR: 100,
      ENODATA: 116,
      ETIME: 117,
      ENOSR: 118,
      ENONET: 119,
      ENOPKG: 120,
      EREMOTE: 121,
      ENOLINK: 47,
      EADV: 122,
      ESRMNT: 123,
      ECOMM: 124,
      EPROTO: 65,
      EMULTIHOP: 36,
      EDOTDOT: 125,
      EBADMSG: 9,
      ENOTUNIQ: 126,
      EBADFD: 127,
      EREMCHG: 128,
      ELIBACC: 129,
      ELIBBAD: 130,
      ELIBSCN: 131,
      ELIBMAX: 132,
      ELIBEXEC: 133,
      ENOSYS: 52,
      ENOTEMPTY: 55,
      ENAMETOOLONG: 37,
      ELOOP: 32,
      EOPNOTSUPP: 138,
      EPFNOSUPPORT: 139,
      ECONNRESET: 15,
      ENOBUFS: 42,
      EAFNOSUPPORT: 5,
      EPROTOTYPE: 67,
      ENOTSOCK: 57,
      ENOPROTOOPT: 50,
      ESHUTDOWN: 140,
      ECONNREFUSED: 14,
      EADDRINUSE: 3,
      ECONNABORTED: 13,
      ENETUNREACH: 40,
      ENETDOWN: 38,
      ETIMEDOUT: 73,
      EHOSTDOWN: 142,
      EHOSTUNREACH: 23,
      EINPROGRESS: 26,
      EALREADY: 7,
      EDESTADDRREQ: 17,
      EMSGSIZE: 35,
      EPROTONOSUPPORT: 66,
      ESOCKTNOSUPPORT: 137,
      EADDRNOTAVAIL: 4,
      ENETRESET: 39,
      EISCONN: 30,
      ENOTCONN: 53,
      ETOOMANYREFS: 141,
      EUSERS: 136,
      EDQUOT: 19,
      ESTALE: 72,
      ENOTSUP: 138,
      ENOMEDIUM: 148,
      EILSEQ: 25,
      EOVERFLOW: 61,
      ECANCELED: 11,
      ENOTRECOVERABLE: 56,
      EOWNERDEAD: 62,
      ESTRPIPE: 135,
    };
    var FS = {
      root: null,
      mounts: [],
      devices: {},
      streams: [],
      nextInode: 1,
      nameTable: null,
      currentPath: '/',
      initialized: false,
      ignorePermissions: true,
      ErrnoError: class extends Error {
        // We set the `name` property to be able to identify `FS.ErrnoError`
        // - the `name` is a standard ECMA-262 property of error objects. Kind of good to have it anyway.
        // - when using PROXYFS, an error can come from an underlying FS
        // as different FS objects have their own FS.ErrnoError each,
        // the test `err instanceof FS.ErrnoError` won't detect an error coming from another filesystem, causing bugs.
        // we'll use the reliable test `err.name == "ErrnoError"` instead
        constructor(errno) {
          super(runtimeInitialized ? strError(errno) : '');
          // TODO(sbc): Use the inline member declaration syntax once we
          // support it in acorn and closure.
          this.name = 'ErrnoError';
          this.errno = errno;
          for (var key in ERRNO_CODES) {
            if (ERRNO_CODES[key] === errno) {
              this.code = key;
              break;
            }
          }
        }
      },
      genericErrors: {},
      filesystems: null,
      syncFSRequests: 0,
      readFiles: {},
      FSStream: class {
        constructor() {
          // TODO(https://github.com/emscripten-core/emscripten/issues/21414):
          // Use inline field declarations.
          this.shared = {};
        }
        get object() {
          return this.node;
        }
        set object(val) {
          this.node = val;
        }
        get isRead() {
          return (this.flags & 2097155) !== 1;
        }
        get isWrite() {
          return (this.flags & 2097155) !== 0;
        }
        get isAppend() {
          return this.flags & 1024;
        }
        get flags() {
          return this.shared.flags;
        }
        set flags(val) {
          this.shared.flags = val;
        }
        get position() {
          return this.shared.position;
        }
        set position(val) {
          this.shared.position = val;
        }
      },
      FSNode: class {
        constructor(parent, name, mode, rdev) {
          if (!parent) {
            parent = this; // root node sets parent to itself
          }
          this.parent = parent;
          this.mount = parent.mount;
          this.mounted = null;
          this.id = FS.nextInode++;
          this.name = name;
          this.mode = mode;
          this.node_ops = {};
          this.stream_ops = {};
          this.rdev = rdev;
          this.readMode = 292 | 73;
          this.writeMode = 146;
        }
        get read() {
          return (this.mode & this.readMode) === this.readMode;
        }
        set read(val) {
          val ? (this.mode |= this.readMode) : (this.mode &= ~this.readMode);
        }
        get write() {
          return (this.mode & this.writeMode) === this.writeMode;
        }
        set write(val) {
          val ? (this.mode |= this.writeMode) : (this.mode &= ~this.writeMode);
        }
        get isFolder() {
          return FS.isDir(this.mode);
        }
        get isDevice() {
          return FS.isChrdev(this.mode);
        }
      },
      lookupPath(path, opts = {}) {
        path = PATH_FS.resolve(path);

        if (!path) return { path: '', node: null };

        var defaults = {
          follow_mount: true,
          recurse_count: 0,
        };
        opts = Object.assign(defaults, opts);

        if (opts.recurse_count > 8) {
          // max recursive lookup of 8
          throw new FS.ErrnoError(32);
        }

        // split the absolute path
        var parts = path.split('/').filter((p) => !!p);

        // start at the root
        var current = FS.root;
        var current_path = '/';

        for (var i = 0; i < parts.length; i++) {
          var islast = i === parts.length - 1;
          if (islast && opts.parent) {
            // stop resolving
            break;
          }

          current = FS.lookupNode(current, parts[i]);
          current_path = PATH.join2(current_path, parts[i]);

          // jump to the mount's root node if this is a mountpoint
          if (FS.isMountpoint(current)) {
            if (!islast || (islast && opts.follow_mount)) {
              current = current.mounted.root;
            }
          }

          // by default, lookupPath will not follow a symlink if it is the final path component.
          // setting opts.follow = true will override this behavior.
          if (!islast || opts.follow) {
            var count = 0;
            while (FS.isLink(current.mode)) {
              var link = FS.readlink(current_path);
              current_path = PATH_FS.resolve(PATH.dirname(current_path), link);

              var lookup = FS.lookupPath(current_path, { recurse_count: opts.recurse_count + 1 });
              current = lookup.node;

              if (count++ > 40) {
                // limit max consecutive symlinks to 40 (SYMLOOP_MAX).
                throw new FS.ErrnoError(32);
              }
            }
          }
        }

        return { path: current_path, node: current };
      },
      getPath(node) {
        var path;
        while (true) {
          if (FS.isRoot(node)) {
            var mount = node.mount.mountpoint;
            if (!path) return mount;
            return mount[mount.length - 1] !== '/' ? `${mount}/${path}` : mount + path;
          }
          path = path ? `${node.name}/${path}` : node.name;
          node = node.parent;
        }
      },
      hashName(parentid, name) {
        var hash = 0;

        for (var i = 0; i < name.length; i++) {
          hash = ((hash << 5) - hash + name.charCodeAt(i)) | 0;
        }
        return ((parentid + hash) >>> 0) % FS.nameTable.length;
      },
      hashAddNode(node) {
        var hash = FS.hashName(node.parent.id, node.name);
        node.name_next = FS.nameTable[hash];
        FS.nameTable[hash] = node;
      },
      hashRemoveNode(node) {
        var hash = FS.hashName(node.parent.id, node.name);
        if (FS.nameTable[hash] === node) {
          FS.nameTable[hash] = node.name_next;
        } else {
          var current = FS.nameTable[hash];
          while (current) {
            if (current.name_next === node) {
              current.name_next = node.name_next;
              break;
            }
            current = current.name_next;
          }
        }
      },
      lookupNode(parent, name) {
        var errCode = FS.mayLookup(parent);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        var hash = FS.hashName(parent.id, name);
        for (var node = FS.nameTable[hash]; node; node = node.name_next) {
          var nodeName = node.name;
          if (node.parent.id === parent.id && nodeName === name) {
            return node;
          }
        }
        // if we failed to find it in the cache, call into the VFS
        return FS.lookup(parent, name);
      },
      createNode(parent, name, mode, rdev) {
        assert(typeof parent == 'object');
        var node = new FS.FSNode(parent, name, mode, rdev);

        FS.hashAddNode(node);

        return node;
      },
      destroyNode(node) {
        FS.hashRemoveNode(node);
      },
      isRoot(node) {
        return node === node.parent;
      },
      isMountpoint(node) {
        return !!node.mounted;
      },
      isFile(mode) {
        return (mode & 61440) === 32768;
      },
      isDir(mode) {
        return (mode & 61440) === 16384;
      },
      isLink(mode) {
        return (mode & 61440) === 40960;
      },
      isChrdev(mode) {
        return (mode & 61440) === 8192;
      },
      isBlkdev(mode) {
        return (mode & 61440) === 24576;
      },
      isFIFO(mode) {
        return (mode & 61440) === 4096;
      },
      isSocket(mode) {
        return (mode & 49152) === 49152;
      },
      flagsToPermissionString(flag) {
        var perms = ['r', 'w', 'rw'][flag & 3];
        if (flag & 512) {
          perms += 'w';
        }
        return perms;
      },
      nodePermissions(node, perms) {
        if (FS.ignorePermissions) {
          return 0;
        }
        // return 0 if any user, group or owner bits are set.
        if (perms.includes('r') && !(node.mode & 292)) {
          return 2;
        } else if (perms.includes('w') && !(node.mode & 146)) {
          return 2;
        } else if (perms.includes('x') && !(node.mode & 73)) {
          return 2;
        }
        return 0;
      },
      mayLookup(dir) {
        if (!FS.isDir(dir.mode)) return 54;
        var errCode = FS.nodePermissions(dir, 'x');
        if (errCode) return errCode;
        if (!dir.node_ops.lookup) return 2;
        return 0;
      },
      mayCreate(dir, name) {
        try {
          var node = FS.lookupNode(dir, name);
          return 20;
        } catch (e) {}
        return FS.nodePermissions(dir, 'wx');
      },
      mayDelete(dir, name, isdir) {
        var node;
        try {
          node = FS.lookupNode(dir, name);
        } catch (e) {
          return e.errno;
        }
        var errCode = FS.nodePermissions(dir, 'wx');
        if (errCode) {
          return errCode;
        }
        if (isdir) {
          if (!FS.isDir(node.mode)) {
            return 54;
          }
          if (FS.isRoot(node) || FS.getPath(node) === FS.cwd()) {
            return 10;
          }
        } else {
          if (FS.isDir(node.mode)) {
            return 31;
          }
        }
        return 0;
      },
      mayOpen(node, flags) {
        if (!node) {
          return 44;
        }
        if (FS.isLink(node.mode)) {
          return 32;
        } else if (FS.isDir(node.mode)) {
          if (
            FS.flagsToPermissionString(flags) !== 'r' || // opening for write
            flags & 512
          ) {
            // TODO: check for O_SEARCH? (== search for dir only)
            return 31;
          }
        }
        return FS.nodePermissions(node, FS.flagsToPermissionString(flags));
      },
      MAX_OPEN_FDS: 4096,
      nextfd() {
        for (var fd = 0; fd <= FS.MAX_OPEN_FDS; fd++) {
          if (!FS.streams[fd]) {
            return fd;
          }
        }
        throw new FS.ErrnoError(33);
      },
      getStreamChecked(fd) {
        var stream = FS.getStream(fd);
        if (!stream) {
          throw new FS.ErrnoError(8);
        }
        return stream;
      },
      getStream: (fd) => FS.streams[fd],
      createStream(stream, fd = -1) {
        assert(fd >= -1);

        // clone it, so we can return an instance of FSStream
        stream = Object.assign(new FS.FSStream(), stream);
        if (fd == -1) {
          fd = FS.nextfd();
        }
        stream.fd = fd;
        FS.streams[fd] = stream;
        return stream;
      },
      closeStream(fd) {
        FS.streams[fd] = null;
      },
      dupStream(origStream, fd = -1) {
        var stream = FS.createStream(origStream, fd);
        stream.stream_ops?.dup?.(stream);
        return stream;
      },
      chrdev_stream_ops: {
        open(stream) {
          var device = FS.getDevice(stream.node.rdev);
          // override node's stream ops with the device's
          stream.stream_ops = device.stream_ops;
          // forward the open call
          stream.stream_ops.open?.(stream);
        },
        llseek() {
          throw new FS.ErrnoError(70);
        },
      },
      major: (dev) => dev >> 8,
      minor: (dev) => dev & 0xff,
      makedev: (ma, mi) => (ma << 8) | mi,
      registerDevice(dev, ops) {
        FS.devices[dev] = { stream_ops: ops };
      },
      getDevice: (dev) => FS.devices[dev],
      getMounts(mount) {
        var mounts = [];
        var check = [mount];

        while (check.length) {
          var m = check.pop();

          mounts.push(m);

          check.push(...m.mounts);
        }

        return mounts;
      },
      syncfs(populate, callback) {
        if (typeof populate == 'function') {
          callback = populate;
          populate = false;
        }

        FS.syncFSRequests++;

        if (FS.syncFSRequests > 1) {
          err(
            `warning: ${FS.syncFSRequests} FS.syncfs operations in flight at once, probably just doing extra work`,
          );
        }

        var mounts = FS.getMounts(FS.root.mount);
        var completed = 0;

        function doCallback(errCode) {
          assert(FS.syncFSRequests > 0);
          FS.syncFSRequests--;
          return callback(errCode);
        }

        function done(errCode) {
          if (errCode) {
            if (!done.errored) {
              done.errored = true;
              return doCallback(errCode);
            }
            return;
          }
          if (++completed >= mounts.length) {
            doCallback(null);
          }
        }

        // sync all mounts
        mounts.forEach((mount) => {
          if (!mount.type.syncfs) {
            return done(null);
          }
          mount.type.syncfs(mount, populate, done);
        });
      },
      mount(type, opts, mountpoint) {
        if (typeof type == 'string') {
          // The filesystem was not included, and instead we have an error
          // message stored in the variable.
          throw type;
        }
        var root = mountpoint === '/';
        var pseudo = !mountpoint;
        var node;

        if (root && FS.root) {
          throw new FS.ErrnoError(10);
        } else if (!root && !pseudo) {
          var lookup = FS.lookupPath(mountpoint, { follow_mount: false });

          mountpoint = lookup.path; // use the absolute path
          node = lookup.node;

          if (FS.isMountpoint(node)) {
            throw new FS.ErrnoError(10);
          }

          if (!FS.isDir(node.mode)) {
            throw new FS.ErrnoError(54);
          }
        }

        var mount = {
          type,
          opts,
          mountpoint,
          mounts: [],
        };

        // create a root node for the fs
        var mountRoot = type.mount(mount);
        mountRoot.mount = mount;
        mount.root = mountRoot;

        if (root) {
          FS.root = mountRoot;
        } else if (node) {
          // set as a mountpoint
          node.mounted = mount;

          // add the new mount to the current mount's children
          if (node.mount) {
            node.mount.mounts.push(mount);
          }
        }

        return mountRoot;
      },
      unmount(mountpoint) {
        var lookup = FS.lookupPath(mountpoint, { follow_mount: false });

        if (!FS.isMountpoint(lookup.node)) {
          throw new FS.ErrnoError(28);
        }

        // destroy the nodes for this mount, and all its child mounts
        var node = lookup.node;
        var mount = node.mounted;
        var mounts = FS.getMounts(mount);

        Object.keys(FS.nameTable).forEach((hash) => {
          var current = FS.nameTable[hash];

          while (current) {
            var next = current.name_next;

            if (mounts.includes(current.mount)) {
              FS.destroyNode(current);
            }

            current = next;
          }
        });

        // no longer a mountpoint
        node.mounted = null;

        // remove this mount from the child mounts
        var idx = node.mount.mounts.indexOf(mount);
        assert(idx !== -1);
        node.mount.mounts.splice(idx, 1);
      },
      lookup(parent, name) {
        return parent.node_ops.lookup(parent, name);
      },
      mknod(path, mode, dev) {
        var lookup = FS.lookupPath(path, { parent: true });
        var parent = lookup.node;
        var name = PATH.basename(path);
        if (!name || name === '.' || name === '..') {
          throw new FS.ErrnoError(28);
        }
        var errCode = FS.mayCreate(parent, name);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        if (!parent.node_ops.mknod) {
          throw new FS.ErrnoError(63);
        }
        return parent.node_ops.mknod(parent, name, mode, dev);
      },
      create(path, mode) {
        mode = mode !== undefined ? mode : 438 /* 0666 */;
        mode &= 4095;
        mode |= 32768;
        return FS.mknod(path, mode, 0);
      },
      mkdir(path, mode) {
        mode = mode !== undefined ? mode : 511 /* 0777 */;
        mode &= 511 | 512;
        mode |= 16384;
        return FS.mknod(path, mode, 0);
      },
      mkdirTree(path, mode) {
        var dirs = path.split('/');
        var d = '';
        for (var i = 0; i < dirs.length; ++i) {
          if (!dirs[i]) continue;
          d += '/' + dirs[i];
          try {
            FS.mkdir(d, mode);
          } catch (e) {
            if (e.errno != 20) throw e;
          }
        }
      },
      mkdev(path, mode, dev) {
        if (typeof dev == 'undefined') {
          dev = mode;
          mode = 438 /* 0666 */;
        }
        mode |= 8192;
        return FS.mknod(path, mode, dev);
      },
      symlink(oldpath, newpath) {
        if (!PATH_FS.resolve(oldpath)) {
          throw new FS.ErrnoError(44);
        }
        var lookup = FS.lookupPath(newpath, { parent: true });
        var parent = lookup.node;
        if (!parent) {
          throw new FS.ErrnoError(44);
        }
        var newname = PATH.basename(newpath);
        var errCode = FS.mayCreate(parent, newname);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        if (!parent.node_ops.symlink) {
          throw new FS.ErrnoError(63);
        }
        return parent.node_ops.symlink(parent, newname, oldpath);
      },
      rename(old_path, new_path) {
        var old_dirname = PATH.dirname(old_path);
        var new_dirname = PATH.dirname(new_path);
        var old_name = PATH.basename(old_path);
        var new_name = PATH.basename(new_path);
        // parents must exist
        var lookup, old_dir, new_dir;

        // let the errors from non existent directories percolate up
        lookup = FS.lookupPath(old_path, { parent: true });
        old_dir = lookup.node;
        lookup = FS.lookupPath(new_path, { parent: true });
        new_dir = lookup.node;

        if (!old_dir || !new_dir) throw new FS.ErrnoError(44);
        // need to be part of the same mount
        if (old_dir.mount !== new_dir.mount) {
          throw new FS.ErrnoError(75);
        }
        // source must exist
        var old_node = FS.lookupNode(old_dir, old_name);
        // old path should not be an ancestor of the new path
        var relative = PATH_FS.relative(old_path, new_dirname);
        if (relative.charAt(0) !== '.') {
          throw new FS.ErrnoError(28);
        }
        // new path should not be an ancestor of the old path
        relative = PATH_FS.relative(new_path, old_dirname);
        if (relative.charAt(0) !== '.') {
          throw new FS.ErrnoError(55);
        }
        // see if the new path already exists
        var new_node;
        try {
          new_node = FS.lookupNode(new_dir, new_name);
        } catch (e) {
          // not fatal
        }
        // early out if nothing needs to change
        if (old_node === new_node) {
          return;
        }
        // we'll need to delete the old entry
        var isdir = FS.isDir(old_node.mode);
        var errCode = FS.mayDelete(old_dir, old_name, isdir);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        // need delete permissions if we'll be overwriting.
        // need create permissions if new doesn't already exist.
        errCode = new_node
          ? FS.mayDelete(new_dir, new_name, isdir)
          : FS.mayCreate(new_dir, new_name);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        if (!old_dir.node_ops.rename) {
          throw new FS.ErrnoError(63);
        }
        if (FS.isMountpoint(old_node) || (new_node && FS.isMountpoint(new_node))) {
          throw new FS.ErrnoError(10);
        }
        // if we are going to change the parent, check write permissions
        if (new_dir !== old_dir) {
          errCode = FS.nodePermissions(old_dir, 'w');
          if (errCode) {
            throw new FS.ErrnoError(errCode);
          }
        }
        // remove the node from the lookup hash
        FS.hashRemoveNode(old_node);
        // do the underlying fs rename
        try {
          old_dir.node_ops.rename(old_node, new_dir, new_name);
          // update old node (we do this here to avoid each backend
          // needing to)
          old_node.parent = new_dir;
        } catch (e) {
          throw e;
        } finally {
          // add the node back to the hash (in case node_ops.rename
          // changed its name)
          FS.hashAddNode(old_node);
        }
      },
      rmdir(path) {
        var lookup = FS.lookupPath(path, { parent: true });
        var parent = lookup.node;
        var name = PATH.basename(path);
        var node = FS.lookupNode(parent, name);
        var errCode = FS.mayDelete(parent, name, true);
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        if (!parent.node_ops.rmdir) {
          throw new FS.ErrnoError(63);
        }
        if (FS.isMountpoint(node)) {
          throw new FS.ErrnoError(10);
        }
        parent.node_ops.rmdir(parent, name);
        FS.destroyNode(node);
      },
      readdir(path) {
        var lookup = FS.lookupPath(path, { follow: true });
        var node = lookup.node;
        if (!node.node_ops.readdir) {
          throw new FS.ErrnoError(54);
        }
        return node.node_ops.readdir(node);
      },
      unlink(path) {
        var lookup = FS.lookupPath(path, { parent: true });
        var parent = lookup.node;
        if (!parent) {
          throw new FS.ErrnoError(44);
        }
        var name = PATH.basename(path);
        var node = FS.lookupNode(parent, name);
        var errCode = FS.mayDelete(parent, name, false);
        if (errCode) {
          // According to POSIX, we should map EISDIR to EPERM, but
          // we instead do what Linux does (and we must, as we use
          // the musl linux libc).
          throw new FS.ErrnoError(errCode);
        }
        if (!parent.node_ops.unlink) {
          throw new FS.ErrnoError(63);
        }
        if (FS.isMountpoint(node)) {
          throw new FS.ErrnoError(10);
        }
        parent.node_ops.unlink(parent, name);
        FS.destroyNode(node);
      },
      readlink(path) {
        var lookup = FS.lookupPath(path);
        var link = lookup.node;
        if (!link) {
          throw new FS.ErrnoError(44);
        }
        if (!link.node_ops.readlink) {
          throw new FS.ErrnoError(28);
        }
        return PATH_FS.resolve(FS.getPath(link.parent), link.node_ops.readlink(link));
      },
      stat(path, dontFollow) {
        var lookup = FS.lookupPath(path, { follow: !dontFollow });
        var node = lookup.node;
        if (!node) {
          throw new FS.ErrnoError(44);
        }
        if (!node.node_ops.getattr) {
          throw new FS.ErrnoError(63);
        }
        return node.node_ops.getattr(node);
      },
      lstat(path) {
        return FS.stat(path, true);
      },
      chmod(path, mode, dontFollow) {
        var node;
        if (typeof path == 'string') {
          var lookup = FS.lookupPath(path, { follow: !dontFollow });
          node = lookup.node;
        } else {
          node = path;
        }
        if (!node.node_ops.setattr) {
          throw new FS.ErrnoError(63);
        }
        node.node_ops.setattr(node, {
          mode: (mode & 4095) | (node.mode & ~4095),
          timestamp: Date.now(),
        });
      },
      lchmod(path, mode) {
        FS.chmod(path, mode, true);
      },
      fchmod(fd, mode) {
        var stream = FS.getStreamChecked(fd);
        FS.chmod(stream.node, mode);
      },
      chown(path, uid, gid, dontFollow) {
        var node;
        if (typeof path == 'string') {
          var lookup = FS.lookupPath(path, { follow: !dontFollow });
          node = lookup.node;
        } else {
          node = path;
        }
        if (!node.node_ops.setattr) {
          throw new FS.ErrnoError(63);
        }
        node.node_ops.setattr(node, {
          timestamp: Date.now(),
          // we ignore the uid / gid for now
        });
      },
      lchown(path, uid, gid) {
        FS.chown(path, uid, gid, true);
      },
      fchown(fd, uid, gid) {
        var stream = FS.getStreamChecked(fd);
        FS.chown(stream.node, uid, gid);
      },
      truncate(path, len) {
        if (len < 0) {
          throw new FS.ErrnoError(28);
        }
        var node;
        if (typeof path == 'string') {
          var lookup = FS.lookupPath(path, { follow: true });
          node = lookup.node;
        } else {
          node = path;
        }
        if (!node.node_ops.setattr) {
          throw new FS.ErrnoError(63);
        }
        if (FS.isDir(node.mode)) {
          throw new FS.ErrnoError(31);
        }
        if (!FS.isFile(node.mode)) {
          throw new FS.ErrnoError(28);
        }
        var errCode = FS.nodePermissions(node, 'w');
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        node.node_ops.setattr(node, {
          size: len,
          timestamp: Date.now(),
        });
      },
      ftruncate(fd, len) {
        var stream = FS.getStreamChecked(fd);
        if ((stream.flags & 2097155) === 0) {
          throw new FS.ErrnoError(28);
        }
        FS.truncate(stream.node, len);
      },
      utime(path, atime, mtime) {
        var lookup = FS.lookupPath(path, { follow: true });
        var node = lookup.node;
        node.node_ops.setattr(node, {
          timestamp: Math.max(atime, mtime),
        });
      },
      open(path, flags, mode) {
        if (path === '') {
          throw new FS.ErrnoError(44);
        }
        flags = typeof flags == 'string' ? FS_modeStringToFlags(flags) : flags;
        if (flags & 64) {
          mode = typeof mode == 'undefined' ? 438 /* 0666 */ : mode;
          mode = (mode & 4095) | 32768;
        } else {
          mode = 0;
        }
        var node;
        if (typeof path == 'object') {
          node = path;
        } else {
          path = PATH.normalize(path);
          try {
            var lookup = FS.lookupPath(path, {
              follow: !(flags & 131072),
            });
            node = lookup.node;
          } catch (e) {
            // ignore
          }
        }
        // perhaps we need to create the node
        var created = false;
        if (flags & 64) {
          if (node) {
            // if O_CREAT and O_EXCL are set, error out if the node already exists
            if (flags & 128) {
              throw new FS.ErrnoError(20);
            }
          } else {
            // node doesn't exist, try to create it
            node = FS.mknod(path, mode, 0);
            created = true;
          }
        }
        if (!node) {
          throw new FS.ErrnoError(44);
        }
        // can't truncate a device
        if (FS.isChrdev(node.mode)) {
          flags &= ~512;
        }
        // if asked only for a directory, then this must be one
        if (flags & 65536 && !FS.isDir(node.mode)) {
          throw new FS.ErrnoError(54);
        }
        // check permissions, if this is not a file we just created now (it is ok to
        // create and write to a file with read-only permissions; it is read-only
        // for later use)
        if (!created) {
          var errCode = FS.mayOpen(node, flags);
          if (errCode) {
            throw new FS.ErrnoError(errCode);
          }
        }
        // do truncation if necessary
        if (flags & 512 && !created) {
          FS.truncate(node, 0);
        }
        // we've already handled these, don't pass down to the underlying vfs
        flags &= ~(128 | 512 | 131072);

        // register the stream with the filesystem
        var stream = FS.createStream({
          node,
          path: FS.getPath(node), // we want the absolute path to the node
          flags,
          seekable: true,
          position: 0,
          stream_ops: node.stream_ops,
          // used by the file family libc calls (fopen, fwrite, ferror, etc.)
          ungotten: [],
          error: false,
        });
        // call the new stream's open function
        if (stream.stream_ops.open) {
          stream.stream_ops.open(stream);
        }
        if (Module['logReadFiles'] && !(flags & 1)) {
          if (!(path in FS.readFiles)) {
            FS.readFiles[path] = 1;
          }
        }
        return stream;
      },
      close(stream) {
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if (stream.getdents) stream.getdents = null; // free readdir state
        try {
          if (stream.stream_ops.close) {
            stream.stream_ops.close(stream);
          }
        } catch (e) {
          throw e;
        } finally {
          FS.closeStream(stream.fd);
        }
        stream.fd = null;
      },
      isClosed(stream) {
        return stream.fd === null;
      },
      llseek(stream, offset, whence) {
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if (!stream.seekable || !stream.stream_ops.llseek) {
          throw new FS.ErrnoError(70);
        }
        if (whence != 0 && whence != 1 && whence != 2) {
          throw new FS.ErrnoError(28);
        }
        stream.position = stream.stream_ops.llseek(stream, offset, whence);
        stream.ungotten = [];
        return stream.position;
      },
      read(stream, buffer, offset, length, position) {
        assert(offset >= 0);
        if (length < 0 || position < 0) {
          throw new FS.ErrnoError(28);
        }
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if ((stream.flags & 2097155) === 1) {
          throw new FS.ErrnoError(8);
        }
        if (FS.isDir(stream.node.mode)) {
          throw new FS.ErrnoError(31);
        }
        if (!stream.stream_ops.read) {
          throw new FS.ErrnoError(28);
        }
        var seeking = typeof position != 'undefined';
        if (!seeking) {
          position = stream.position;
        } else if (!stream.seekable) {
          throw new FS.ErrnoError(70);
        }
        var bytesRead = stream.stream_ops.read(stream, buffer, offset, length, position);
        if (!seeking) stream.position += bytesRead;
        return bytesRead;
      },
      write(stream, buffer, offset, length, position, canOwn) {
        assert(offset >= 0);
        if (length < 0 || position < 0) {
          throw new FS.ErrnoError(28);
        }
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if ((stream.flags & 2097155) === 0) {
          throw new FS.ErrnoError(8);
        }
        if (FS.isDir(stream.node.mode)) {
          throw new FS.ErrnoError(31);
        }
        if (!stream.stream_ops.write) {
          throw new FS.ErrnoError(28);
        }
        if (stream.seekable && stream.flags & 1024) {
          // seek to the end before writing in append mode
          FS.llseek(stream, 0, 2);
        }
        var seeking = typeof position != 'undefined';
        if (!seeking) {
          position = stream.position;
        } else if (!stream.seekable) {
          throw new FS.ErrnoError(70);
        }
        var bytesWritten = stream.stream_ops.write(
          stream,
          buffer,
          offset,
          length,
          position,
          canOwn,
        );
        if (!seeking) stream.position += bytesWritten;
        return bytesWritten;
      },
      allocate(stream, offset, length) {
        if (FS.isClosed(stream)) {
          throw new FS.ErrnoError(8);
        }
        if (offset < 0 || length <= 0) {
          throw new FS.ErrnoError(28);
        }
        if ((stream.flags & 2097155) === 0) {
          throw new FS.ErrnoError(8);
        }
        if (!FS.isFile(stream.node.mode) && !FS.isDir(stream.node.mode)) {
          throw new FS.ErrnoError(43);
        }
        if (!stream.stream_ops.allocate) {
          throw new FS.ErrnoError(138);
        }
        stream.stream_ops.allocate(stream, offset, length);
      },
      mmap(stream, length, position, prot, flags) {
        // User requests writing to file (prot & PROT_WRITE != 0).
        // Checking if we have permissions to write to the file unless
        // MAP_PRIVATE flag is set. According to POSIX spec it is possible
        // to write to file opened in read-only mode with MAP_PRIVATE flag,
        // as all modifications will be visible only in the memory of
        // the current process.
        if ((prot & 2) !== 0 && (flags & 2) === 0 && (stream.flags & 2097155) !== 2) {
          throw new FS.ErrnoError(2);
        }
        if ((stream.flags & 2097155) === 1) {
          throw new FS.ErrnoError(2);
        }
        if (!stream.stream_ops.mmap) {
          throw new FS.ErrnoError(43);
        }
        if (!length) {
          throw new FS.ErrnoError(28);
        }
        return stream.stream_ops.mmap(stream, length, position, prot, flags);
      },
      msync(stream, buffer, offset, length, mmapFlags) {
        assert(offset >= 0);
        if (!stream.stream_ops.msync) {
          return 0;
        }
        return stream.stream_ops.msync(stream, buffer, offset, length, mmapFlags);
      },
      ioctl(stream, cmd, arg) {
        if (!stream.stream_ops.ioctl) {
          throw new FS.ErrnoError(59);
        }
        return stream.stream_ops.ioctl(stream, cmd, arg);
      },
      readFile(path, opts = {}) {
        opts.flags = opts.flags || 0;
        opts.encoding = opts.encoding || 'binary';
        if (opts.encoding !== 'utf8' && opts.encoding !== 'binary') {
          throw new Error(`Invalid encoding type "${opts.encoding}"`);
        }
        var ret;
        var stream = FS.open(path, opts.flags);
        var stat = FS.stat(path);
        var length = stat.size;
        var buf = new Uint8Array(length);
        FS.read(stream, buf, 0, length, 0);
        if (opts.encoding === 'utf8') {
          ret = UTF8ArrayToString(buf);
        } else if (opts.encoding === 'binary') {
          ret = buf;
        }
        FS.close(stream);
        return ret;
      },
      writeFile(path, data, opts = {}) {
        opts.flags = opts.flags || 577;
        var stream = FS.open(path, opts.flags, opts.mode);
        if (typeof data == 'string') {
          var buf = new Uint8Array(lengthBytesUTF8(data) + 1);
          var actualNumBytes = stringToUTF8Array(data, buf, 0, buf.length);
          FS.write(stream, buf, 0, actualNumBytes, undefined, opts.canOwn);
        } else if (ArrayBuffer.isView(data)) {
          FS.write(stream, data, 0, data.byteLength, undefined, opts.canOwn);
        } else {
          throw new Error('Unsupported data type');
        }
        FS.close(stream);
      },
      cwd: () => FS.currentPath,
      chdir(path) {
        var lookup = FS.lookupPath(path, { follow: true });
        if (lookup.node === null) {
          throw new FS.ErrnoError(44);
        }
        if (!FS.isDir(lookup.node.mode)) {
          throw new FS.ErrnoError(54);
        }
        var errCode = FS.nodePermissions(lookup.node, 'x');
        if (errCode) {
          throw new FS.ErrnoError(errCode);
        }
        FS.currentPath = lookup.path;
      },
      createDefaultDirectories() {
        FS.mkdir('/tmp');
        FS.mkdir('/home');
        FS.mkdir('/home/web_user');
      },
      createDefaultDevices() {
        // create /dev
        FS.mkdir('/dev');
        // setup /dev/null
        FS.registerDevice(FS.makedev(1, 3), {
          read: () => 0,
          write: (stream, buffer, offset, length, pos) => length,
        });
        FS.mkdev('/dev/null', FS.makedev(1, 3));
        // setup /dev/tty and /dev/tty1
        // stderr needs to print output using err() rather than out()
        // so we register a second tty just for it.
        TTY.register(FS.makedev(5, 0), TTY.default_tty_ops);
        TTY.register(FS.makedev(6, 0), TTY.default_tty1_ops);
        FS.mkdev('/dev/tty', FS.makedev(5, 0));
        FS.mkdev('/dev/tty1', FS.makedev(6, 0));
        // setup /dev/[u]random
        // use a buffer to avoid overhead of individual crypto calls per byte
        var randomBuffer = new Uint8Array(1024),
          randomLeft = 0;
        var randomByte = () => {
          if (randomLeft === 0) {
            randomLeft = randomFill(randomBuffer).byteLength;
          }
          return randomBuffer[--randomLeft];
        };
        FS.createDevice('/dev', 'random', randomByte);
        FS.createDevice('/dev', 'urandom', randomByte);
        // we're not going to emulate the actual shm device,
        // just create the tmp dirs that reside in it commonly
        FS.mkdir('/dev/shm');
        FS.mkdir('/dev/shm/tmp');
      },
      createSpecialDirectories() {
        // create /proc/self/fd which allows /proc/self/fd/6 => readlink gives the
        // name of the stream for fd 6 (see test_unistd_ttyname)
        FS.mkdir('/proc');
        var proc_self = FS.mkdir('/proc/self');
        FS.mkdir('/proc/self/fd');
        FS.mount(
          {
            mount() {
              var node = FS.createNode(proc_self, 'fd', 16384 | 511 /* 0777 */, 73);
              node.node_ops = {
                lookup(parent, name) {
                  var fd = +name;
                  var stream = FS.getStreamChecked(fd);
                  var ret = {
                    parent: null,
                    mount: { mountpoint: 'fake' },
                    node_ops: { readlink: () => stream.path },
                  };
                  ret.parent = ret; // make it look like a simple root node
                  return ret;
                },
              };
              return node;
            },
          },
          {},
          '/proc/self/fd',
        );
      },
      createStandardStreams(input, output, error) {
        // TODO deprecate the old functionality of a single
        // input / output callback and that utilizes FS.createDevice
        // and instead require a unique set of stream ops

        // by default, we symlink the standard streams to the
        // default tty devices. however, if the standard streams
        // have been overwritten we create a unique device for
        // them instead.
        if (input) {
          FS.createDevice('/dev', 'stdin', input);
        } else {
          FS.symlink('/dev/tty', '/dev/stdin');
        }
        if (output) {
          FS.createDevice('/dev', 'stdout', null, output);
        } else {
          FS.symlink('/dev/tty', '/dev/stdout');
        }
        if (error) {
          FS.createDevice('/dev', 'stderr', null, error);
        } else {
          FS.symlink('/dev/tty1', '/dev/stderr');
        }

        // open default streams for the stdin, stdout and stderr devices
        var stdin = FS.open('/dev/stdin', 0);
        var stdout = FS.open('/dev/stdout', 1);
        var stderr = FS.open('/dev/stderr', 1);
        assert(stdin.fd === 0, `invalid handle for stdin (${stdin.fd})`);
        assert(stdout.fd === 1, `invalid handle for stdout (${stdout.fd})`);
        assert(stderr.fd === 2, `invalid handle for stderr (${stderr.fd})`);
      },
      staticInit() {
        // Some errors may happen quite a bit, to avoid overhead we reuse them (and suffer a lack of stack info)
        [44].forEach((code) => {
          FS.genericErrors[code] = new FS.ErrnoError(code);
          FS.genericErrors[code].stack = '<generic error, no stack>';
        });

        FS.nameTable = new Array(4096);

        FS.mount(MEMFS, {}, '/');

        FS.createDefaultDirectories();
        FS.createDefaultDevices();
        FS.createSpecialDirectories();

        FS.filesystems = {
          MEMFS: MEMFS,
        };
      },
      init(input, output, error) {
        assert(
          !FS.initialized,
          'FS.init was previously called. If you want to initialize later with custom parameters, remove any earlier calls (note that one is automatically added to the generated code)',
        );
        FS.initialized = true;

        // Allow Module.stdin etc. to provide defaults, if none explicitly passed to us here
        input ??= Module['stdin'];
        output ??= Module['stdout'];
        error ??= Module['stderr'];

        FS.createStandardStreams(input, output, error);
      },
      quit() {
        FS.initialized = false;
        // force-flush all streams, so we get musl std streams printed out
        _fflush(0);
        // close all of our streams
        for (var i = 0; i < FS.streams.length; i++) {
          var stream = FS.streams[i];
          if (!stream) {
            continue;
          }
          FS.close(stream);
        }
      },
      findObject(path, dontResolveLastLink) {
        var ret = FS.analyzePath(path, dontResolveLastLink);
        if (!ret.exists) {
          return null;
        }
        return ret.object;
      },
      analyzePath(path, dontResolveLastLink) {
        // operate from within the context of the symlink's target
        try {
          var lookup = FS.lookupPath(path, { follow: !dontResolveLastLink });
          path = lookup.path;
        } catch (e) {}
        var ret = {
          isRoot: false,
          exists: false,
          error: 0,
          name: null,
          path: null,
          object: null,
          parentExists: false,
          parentPath: null,
          parentObject: null,
        };
        try {
          var lookup = FS.lookupPath(path, { parent: true });
          ret.parentExists = true;
          ret.parentPath = lookup.path;
          ret.parentObject = lookup.node;
          ret.name = PATH.basename(path);
          lookup = FS.lookupPath(path, { follow: !dontResolveLastLink });
          ret.exists = true;
          ret.path = lookup.path;
          ret.object = lookup.node;
          ret.name = lookup.node.name;
          ret.isRoot = lookup.path === '/';
        } catch (e) {
          ret.error = e.errno;
        }
        return ret;
      },
      createPath(parent, path, canRead, canWrite) {
        parent = typeof parent == 'string' ? parent : FS.getPath(parent);
        var parts = path.split('/').reverse();
        while (parts.length) {
          var part = parts.pop();
          if (!part) continue;
          var current = PATH.join2(parent, part);
          try {
            FS.mkdir(current);
          } catch (e) {
            // ignore EEXIST
          }
          parent = current;
        }
        return current;
      },
      createFile(parent, name, properties, canRead, canWrite) {
        var path = PATH.join2(typeof parent == 'string' ? parent : FS.getPath(parent), name);
        var mode = FS_getMode(canRead, canWrite);
        return FS.create(path, mode);
      },
      createDataFile(parent, name, data, canRead, canWrite, canOwn) {
        var path = name;
        if (parent) {
          parent = typeof parent == 'string' ? parent : FS.getPath(parent);
          path = name ? PATH.join2(parent, name) : parent;
        }
        var mode = FS_getMode(canRead, canWrite);
        var node = FS.create(path, mode);
        if (data) {
          if (typeof data == 'string') {
            var arr = new Array(data.length);
            for (var i = 0, len = data.length; i < len; ++i) arr[i] = data.charCodeAt(i);
            data = arr;
          }
          // make sure we can write to the file
          FS.chmod(node, mode | 146);
          var stream = FS.open(node, 577);
          FS.write(stream, data, 0, data.length, 0, canOwn);
          FS.close(stream);
          FS.chmod(node, mode);
        }
      },
      createDevice(parent, name, input, output) {
        var path = PATH.join2(typeof parent == 'string' ? parent : FS.getPath(parent), name);
        var mode = FS_getMode(!!input, !!output);
        FS.createDevice.major ??= 64;
        var dev = FS.makedev(FS.createDevice.major++, 0);
        // Create a fake device that a set of stream ops to emulate
        // the old behavior.
        FS.registerDevice(dev, {
          open(stream) {
            stream.seekable = false;
          },
          close(stream) {
            // flush any pending line data
            if (output?.buffer?.length) {
              output(10);
            }
          },
          read(stream, buffer, offset, length, pos /* ignored */) {
            var bytesRead = 0;
            for (var i = 0; i < length; i++) {
              var result;
              try {
                result = input();
              } catch (e) {
                throw new FS.ErrnoError(29);
              }
              if (result === undefined && bytesRead === 0) {
                throw new FS.ErrnoError(6);
              }
              if (result === null || result === undefined) break;
              bytesRead++;
              buffer[offset + i] = result;
            }
            if (bytesRead) {
              stream.node.timestamp = Date.now();
            }
            return bytesRead;
          },
          write(stream, buffer, offset, length, pos) {
            for (var i = 0; i < length; i++) {
              try {
                output(buffer[offset + i]);
              } catch (e) {
                throw new FS.ErrnoError(29);
              }
            }
            if (length) {
              stream.node.timestamp = Date.now();
            }
            return i;
          },
        });
        return FS.mkdev(path, mode, dev);
      },
      forceLoadFile(obj) {
        if (obj.isDevice || obj.isFolder || obj.link || obj.contents) return true;
        if (typeof XMLHttpRequest != 'undefined') {
          throw new Error(
            'Lazy loading should have been performed (contents set) in createLazyFile, but it was not. Lazy loading only works in web workers. Use --embed-file or --preload-file in emcc on the main thread.',
          );
        } else {
          // Command-line.
          try {
            obj.contents = readBinary(obj.url);
            obj.usedBytes = obj.contents.length;
          } catch (e) {
            throw new FS.ErrnoError(29);
          }
        }
      },
      createLazyFile(parent, name, url, canRead, canWrite) {
        // Lazy chunked Uint8Array (implements get and length from Uint8Array).
        // Actual getting is abstracted away for eventual reuse.
        class LazyUint8Array {
          constructor() {
            this.lengthKnown = false;
            this.chunks = []; // Loaded chunks. Index is the chunk number
          }
          get(idx) {
            if (idx > this.length - 1 || idx < 0) {
              return undefined;
            }
            var chunkOffset = idx % this.chunkSize;
            var chunkNum = (idx / this.chunkSize) | 0;
            return this.getter(chunkNum)[chunkOffset];
          }
          setDataGetter(getter) {
            this.getter = getter;
          }
          cacheLength() {
            // Find length
            var xhr = new XMLHttpRequest();
            xhr.open('HEAD', url, false);
            xhr.send(null);
            if (!((xhr.status >= 200 && xhr.status < 300) || xhr.status === 304))
              throw new Error("Couldn't load " + url + '. Status: ' + xhr.status);
            var datalength = Number(xhr.getResponseHeader('Content-length'));
            var header;
            var hasByteServing =
              (header = xhr.getResponseHeader('Accept-Ranges')) && header === 'bytes';
            var usesGzip =
              (header = xhr.getResponseHeader('Content-Encoding')) && header === 'gzip';

            var chunkSize = 1024 * 1024; // Chunk size in bytes

            if (!hasByteServing) chunkSize = datalength;

            // Function to get a range from the remote URL.
            var doXHR = (from, to) => {
              if (from > to)
                throw new Error('invalid range (' + from + ', ' + to + ') or no bytes requested!');
              if (to > datalength - 1)
                throw new Error('only ' + datalength + ' bytes available! programmer error!');

              // TODO: Use mozResponseArrayBuffer, responseStream, etc. if available.
              var xhr = new XMLHttpRequest();
              xhr.open('GET', url, false);
              if (datalength !== chunkSize)
                xhr.setRequestHeader('Range', 'bytes=' + from + '-' + to);

              // Some hints to the browser that we want binary data.
              xhr.responseType = 'arraybuffer';
              if (xhr.overrideMimeType) {
                xhr.overrideMimeType('text/plain; charset=x-user-defined');
              }

              xhr.send(null);
              if (!((xhr.status >= 200 && xhr.status < 300) || xhr.status === 304))
                throw new Error("Couldn't load " + url + '. Status: ' + xhr.status);
              if (xhr.response !== undefined) {
                return new Uint8Array(/** @type{Array<number>} */ (xhr.response || []));
              }
              return intArrayFromString(xhr.responseText || '', true);
            };
            var lazyArray = this;
            lazyArray.setDataGetter((chunkNum) => {
              var start = chunkNum * chunkSize;
              var end = (chunkNum + 1) * chunkSize - 1; // including this byte
              end = Math.min(end, datalength - 1); // if datalength-1 is selected, this is the last block
              if (typeof lazyArray.chunks[chunkNum] == 'undefined') {
                lazyArray.chunks[chunkNum] = doXHR(start, end);
              }
              if (typeof lazyArray.chunks[chunkNum] == 'undefined')
                throw new Error('doXHR failed!');
              return lazyArray.chunks[chunkNum];
            });

            if (usesGzip || !datalength) {
              // if the server uses gzip or doesn't supply the length, we have to download the whole file to get the (uncompressed) length
              chunkSize = datalength = 1; // this will force getter(0)/doXHR do download the whole file
              datalength = this.getter(0).length;
              chunkSize = datalength;
              out('LazyFiles on gzip forces download of the whole file when length is accessed');
            }

            this._length = datalength;
            this._chunkSize = chunkSize;
            this.lengthKnown = true;
          }
          get length() {
            if (!this.lengthKnown) {
              this.cacheLength();
            }
            return this._length;
          }
          get chunkSize() {
            if (!this.lengthKnown) {
              this.cacheLength();
            }
            return this._chunkSize;
          }
        }

        if (typeof XMLHttpRequest != 'undefined') {
          if (!ENVIRONMENT_IS_WORKER)
            throw 'Cannot do synchronous binary XHRs outside webworkers in modern browsers. Use --embed-file or --preload-file in emcc';
          var lazyArray = new LazyUint8Array();
          var properties = { isDevice: false, contents: lazyArray };
        } else {
          var properties = { isDevice: false, url: url };
        }

        var node = FS.createFile(parent, name, properties, canRead, canWrite);
        // This is a total hack, but I want to get this lazy file code out of the
        // core of MEMFS. If we want to keep this lazy file concept I feel it should
        // be its own thin LAZYFS proxying calls to MEMFS.
        if (properties.contents) {
          node.contents = properties.contents;
        } else if (properties.url) {
          node.contents = null;
          node.url = properties.url;
        }
        // Add a function that defers querying the file size until it is asked the first time.
        Object.defineProperties(node, {
          usedBytes: {
            get: function () {
              return this.contents.length;
            },
          },
        });
        // override each stream op with one that tries to force load the lazy file first
        var stream_ops = {};
        var keys = Object.keys(node.stream_ops);
        keys.forEach((key) => {
          var fn = node.stream_ops[key];
          stream_ops[key] = (...args) => {
            FS.forceLoadFile(node);
            return fn(...args);
          };
        });
        function writeChunks(stream, buffer, offset, length, position) {
          var contents = stream.node.contents;
          if (position >= contents.length) return 0;
          var size = Math.min(contents.length - position, length);
          assert(size >= 0);
          if (contents.slice) {
            // normal array
            for (var i = 0; i < size; i++) {
              buffer[offset + i] = contents[position + i];
            }
          } else {
            for (var i = 0; i < size; i++) {
              // LazyUint8Array from sync binary XHR
              buffer[offset + i] = contents.get(position + i);
            }
          }
          return size;
        }
        // use a custom read function
        stream_ops.read = (stream, buffer, offset, length, position) => {
          FS.forceLoadFile(node);
          return writeChunks(stream, buffer, offset, length, position);
        };
        // use a custom mmap function
        stream_ops.mmap = (stream, length, position, prot, flags) => {
          FS.forceLoadFile(node);
          var ptr = mmapAlloc(length);
          if (!ptr) {
            throw new FS.ErrnoError(48);
          }
          writeChunks(stream, HEAP8, ptr, length, position);
          return { ptr, allocated: true };
        };
        node.stream_ops = stream_ops;
        return node;
      },
      absolutePath() {
        abort('FS.absolutePath has been removed; use PATH_FS.resolve instead');
      },
      createFolder() {
        abort('FS.createFolder has been removed; use FS.mkdir instead');
      },
      createLink() {
        abort('FS.createLink has been removed; use FS.symlink instead');
      },
      joinPath() {
        abort('FS.joinPath has been removed; use PATH.join instead');
      },
      mmapAlloc() {
        abort('FS.mmapAlloc has been replaced by the top level function mmapAlloc');
      },
      standardizePath() {
        abort('FS.standardizePath has been removed; use PATH.normalize instead');
      },
    };

    var SYSCALLS = {
      DEFAULT_POLLMASK: 5,
      calculateAt(dirfd, path, allowEmpty) {
        if (PATH.isAbs(path)) {
          return path;
        }
        // relative path
        var dir;
        if (dirfd === -100) {
          dir = FS.cwd();
        } else {
          var dirstream = SYSCALLS.getStreamFromFD(dirfd);
          dir = dirstream.path;
        }
        if (path.length == 0) {
          if (!allowEmpty) {
            throw new FS.ErrnoError(44);
          }
          return dir;
        }
        return PATH.join2(dir, path);
      },
      doStat(func, path, buf) {
        var stat = func(path);
        HEAP32[buf >> 2] = stat.dev;
        HEAP32[(buf + 4) >> 2] = stat.mode;
        HEAPU32[(buf + 8) >> 2] = stat.nlink;
        HEAP32[(buf + 12) >> 2] = stat.uid;
        HEAP32[(buf + 16) >> 2] = stat.gid;
        HEAP32[(buf + 20) >> 2] = stat.rdev;
        (tempI64 = [
          stat.size >>> 0,
          ((tempDouble = stat.size),
          +Math.abs(tempDouble) >= 1.0
            ? tempDouble > 0.0
              ? +Math.floor(tempDouble / 4294967296.0) >>> 0
              : ~~+Math.ceil((tempDouble - +(~~tempDouble >>> 0)) / 4294967296.0) >>> 0
            : 0),
        ]),
          (HEAP32[(buf + 24) >> 2] = tempI64[0]),
          (HEAP32[(buf + 28) >> 2] = tempI64[1]);
        HEAP32[(buf + 32) >> 2] = 4096;
        HEAP32[(buf + 36) >> 2] = stat.blocks;
        var atime = stat.atime.getTime();
        var mtime = stat.mtime.getTime();
        var ctime = stat.ctime.getTime();
        (tempI64 = [
          Math.floor(atime / 1000) >>> 0,
          ((tempDouble = Math.floor(atime / 1000)),
          +Math.abs(tempDouble) >= 1.0
            ? tempDouble > 0.0
              ? +Math.floor(tempDouble / 4294967296.0) >>> 0
              : ~~+Math.ceil((tempDouble - +(~~tempDouble >>> 0)) / 4294967296.0) >>> 0
            : 0),
        ]),
          (HEAP32[(buf + 40) >> 2] = tempI64[0]),
          (HEAP32[(buf + 44) >> 2] = tempI64[1]);
        HEAPU32[(buf + 48) >> 2] = (atime % 1000) * 1000 * 1000;
        (tempI64 = [
          Math.floor(mtime / 1000) >>> 0,
          ((tempDouble = Math.floor(mtime / 1000)),
          +Math.abs(tempDouble) >= 1.0
            ? tempDouble > 0.0
              ? +Math.floor(tempDouble / 4294967296.0) >>> 0
              : ~~+Math.ceil((tempDouble - +(~~tempDouble >>> 0)) / 4294967296.0) >>> 0
            : 0),
        ]),
          (HEAP32[(buf + 56) >> 2] = tempI64[0]),
          (HEAP32[(buf + 60) >> 2] = tempI64[1]);
        HEAPU32[(buf + 64) >> 2] = (mtime % 1000) * 1000 * 1000;
        (tempI64 = [
          Math.floor(ctime / 1000) >>> 0,
          ((tempDouble = Math.floor(ctime / 1000)),
          +Math.abs(tempDouble) >= 1.0
            ? tempDouble > 0.0
              ? +Math.floor(tempDouble / 4294967296.0) >>> 0
              : ~~+Math.ceil((tempDouble - +(~~tempDouble >>> 0)) / 4294967296.0) >>> 0
            : 0),
        ]),
          (HEAP32[(buf + 72) >> 2] = tempI64[0]),
          (HEAP32[(buf + 76) >> 2] = tempI64[1]);
        HEAPU32[(buf + 80) >> 2] = (ctime % 1000) * 1000 * 1000;
        (tempI64 = [
          stat.ino >>> 0,
          ((tempDouble = stat.ino),
          +Math.abs(tempDouble) >= 1.0
            ? tempDouble > 0.0
              ? +Math.floor(tempDouble / 4294967296.0) >>> 0
              : ~~+Math.ceil((tempDouble - +(~~tempDouble >>> 0)) / 4294967296.0) >>> 0
            : 0),
        ]),
          (HEAP32[(buf + 88) >> 2] = tempI64[0]),
          (HEAP32[(buf + 92) >> 2] = tempI64[1]);
        return 0;
      },
      doMsync(addr, stream, len, flags, offset) {
        if (!FS.isFile(stream.node.mode)) {
          throw new FS.ErrnoError(43);
        }
        if (flags & 2) {
          // MAP_PRIVATE calls need not to be synced back to underlying fs
          return 0;
        }
        var buffer = HEAPU8.slice(addr, addr + len);
        FS.msync(stream, buffer, offset, len, flags);
      },
      getStreamFromFD(fd) {
        var stream = FS.getStreamChecked(fd);
        return stream;
      },
      varargs: undefined,
      getStr(ptr) {
        var ret = UTF8ToString(ptr);
        return ret;
      },
    };
    function ___syscall_fcntl64(fd, cmd, varargs) {
      SYSCALLS.varargs = varargs;
      try {
        var stream = SYSCALLS.getStreamFromFD(fd);
        switch (cmd) {
          case 0: {
            var arg = syscallGetVarargI();
            if (arg < 0) {
              return -28;
            }
            while (FS.streams[arg]) {
              arg++;
            }
            var newStream;
            newStream = FS.dupStream(stream, arg);
            return newStream.fd;
          }
          case 1:
          case 2:
            return 0; // FD_CLOEXEC makes no sense for a single process.
          case 3:
            return stream.flags;
          case 4: {
            var arg = syscallGetVarargI();
            stream.flags |= arg;
            return 0;
          }
          case 12: {
            var arg = syscallGetVarargP();
            var offset = 0;
            // We're always unlocked.
            HEAP16[(arg + offset) >> 1] = 2;
            return 0;
          }
          case 13:
          case 14:
            return 0; // Pretend that the locking is successful.
        }
        return -28;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    function ___syscall_fstat64(fd, buf) {
      try {
        var stream = SYSCALLS.getStreamFromFD(fd);
        return SYSCALLS.doStat(FS.stat, stream.path, buf);
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    var convertI32PairToI53Checked = (lo, hi) => {
      assert(lo == lo >>> 0 || lo == (lo | 0)); // lo should either be a i32 or a u32
      assert(hi === (hi | 0)); // hi should be a i32
      return (hi + 0x200000) >>> 0 < 0x400001 - !!lo ? (lo >>> 0) + hi * 4294967296 : NaN;
    };
    function ___syscall_ftruncate64(fd, length_low, length_high) {
      var length = convertI32PairToI53Checked(length_low, length_high);

      try {
        if (isNaN(length)) return 61;
        FS.ftruncate(fd, length);
        return 0;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    var stringToUTF8 = (str, outPtr, maxBytesToWrite) => {
      assert(
        typeof maxBytesToWrite == 'number',
        'stringToUTF8(str, outPtr, maxBytesToWrite) is missing the third parameter that specifies the length of the output buffer!',
      );
      return stringToUTF8Array(str, HEAPU8, outPtr, maxBytesToWrite);
    };

    function ___syscall_getdents64(fd, dirp, count) {
      try {
        var stream = SYSCALLS.getStreamFromFD(fd);
        stream.getdents ||= FS.readdir(stream.path);

        var struct_size = 280;
        var pos = 0;
        var off = FS.llseek(stream, 0, 1);

        var idx = Math.floor(off / struct_size);

        while (idx < stream.getdents.length && pos + struct_size <= count) {
          var id;
          var type;
          var name = stream.getdents[idx];
          if (name === '.') {
            id = stream.node.id;
            type = 4; // DT_DIR
          } else if (name === '..') {
            var lookup = FS.lookupPath(stream.path, { parent: true });
            id = lookup.node.id;
            type = 4; // DT_DIR
          } else {
            var child = FS.lookupNode(stream.node, name);
            id = child.id;
            type = FS.isChrdev(child.mode)
              ? 2 // DT_CHR, character device.
              : FS.isDir(child.mode)
                ? 4 // DT_DIR, directory.
                : FS.isLink(child.mode)
                  ? 10 // DT_LNK, symbolic link.
                  : 8; // DT_REG, regular file.
          }
          assert(id);
          (tempI64 = [
            id >>> 0,
            ((tempDouble = id),
            +Math.abs(tempDouble) >= 1.0
              ? tempDouble > 0.0
                ? +Math.floor(tempDouble / 4294967296.0) >>> 0
                : ~~+Math.ceil((tempDouble - +(~~tempDouble >>> 0)) / 4294967296.0) >>> 0
              : 0),
          ]),
            (HEAP32[(dirp + pos) >> 2] = tempI64[0]),
            (HEAP32[(dirp + pos + 4) >> 2] = tempI64[1]);
          (tempI64 = [
            ((idx + 1) * struct_size) >>> 0,
            ((tempDouble = (idx + 1) * struct_size),
            +Math.abs(tempDouble) >= 1.0
              ? tempDouble > 0.0
                ? +Math.floor(tempDouble / 4294967296.0) >>> 0
                : ~~+Math.ceil((tempDouble - +(~~tempDouble >>> 0)) / 4294967296.0) >>> 0
              : 0),
          ]),
            (HEAP32[(dirp + pos + 8) >> 2] = tempI64[0]),
            (HEAP32[(dirp + pos + 12) >> 2] = tempI64[1]);
          HEAP16[(dirp + pos + 16) >> 1] = 280;
          HEAP8[dirp + pos + 18] = type;
          stringToUTF8(name, dirp + pos + 19, 256);
          pos += struct_size;
          idx += 1;
        }
        FS.llseek(stream, idx * struct_size, 0);
        return pos;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    function ___syscall_ioctl(fd, op, varargs) {
      SYSCALLS.varargs = varargs;
      try {
        var stream = SYSCALLS.getStreamFromFD(fd);
        switch (op) {
          case 21509: {
            if (!stream.tty) return -59;
            return 0;
          }
          case 21505: {
            if (!stream.tty) return -59;
            if (stream.tty.ops.ioctl_tcgets) {
              var termios = stream.tty.ops.ioctl_tcgets(stream);
              var argp = syscallGetVarargP();
              HEAP32[argp >> 2] = termios.c_iflag || 0;
              HEAP32[(argp + 4) >> 2] = termios.c_oflag || 0;
              HEAP32[(argp + 8) >> 2] = termios.c_cflag || 0;
              HEAP32[(argp + 12) >> 2] = termios.c_lflag || 0;
              for (var i = 0; i < 32; i++) {
                HEAP8[argp + i + 17] = termios.c_cc[i] || 0;
              }
              return 0;
            }
            return 0;
          }
          case 21510:
          case 21511:
          case 21512: {
            if (!stream.tty) return -59;
            return 0; // no-op, not actually adjusting terminal settings
          }
          case 21506:
          case 21507:
          case 21508: {
            if (!stream.tty) return -59;
            if (stream.tty.ops.ioctl_tcsets) {
              var argp = syscallGetVarargP();
              var c_iflag = HEAP32[argp >> 2];
              var c_oflag = HEAP32[(argp + 4) >> 2];
              var c_cflag = HEAP32[(argp + 8) >> 2];
              var c_lflag = HEAP32[(argp + 12) >> 2];
              var c_cc = [];
              for (var i = 0; i < 32; i++) {
                c_cc.push(HEAP8[argp + i + 17]);
              }
              return stream.tty.ops.ioctl_tcsets(stream.tty, op, {
                c_iflag,
                c_oflag,
                c_cflag,
                c_lflag,
                c_cc,
              });
            }
            return 0; // no-op, not actually adjusting terminal settings
          }
          case 21519: {
            if (!stream.tty) return -59;
            var argp = syscallGetVarargP();
            HEAP32[argp >> 2] = 0;
            return 0;
          }
          case 21520: {
            if (!stream.tty) return -59;
            return -28; // not supported
          }
          case 21531: {
            var argp = syscallGetVarargP();
            return FS.ioctl(stream, op, argp);
          }
          case 21523: {
            // TODO: in theory we should write to the winsize struct that gets
            // passed in, but for now musl doesn't read anything on it
            if (!stream.tty) return -59;
            if (stream.tty.ops.ioctl_tiocgwinsz) {
              var winsize = stream.tty.ops.ioctl_tiocgwinsz(stream.tty);
              var argp = syscallGetVarargP();
              HEAP16[argp >> 1] = winsize[0];
              HEAP16[(argp + 2) >> 1] = winsize[1];
            }
            return 0;
          }
          case 21524: {
            // TODO: technically, this ioctl call should change the window size.
            // but, since emscripten doesn't have any concept of a terminal window
            // yet, we'll just silently throw it away as we do TIOCGWINSZ
            if (!stream.tty) return -59;
            return 0;
          }
          case 21515: {
            if (!stream.tty) return -59;
            return 0;
          }
          default:
            return -28; // not supported
        }
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    function ___syscall_lstat64(path, buf) {
      try {
        path = SYSCALLS.getStr(path);
        return SYSCALLS.doStat(FS.lstat, path, buf);
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    function ___syscall_newfstatat(dirfd, path, buf, flags) {
      try {
        path = SYSCALLS.getStr(path);
        var nofollow = flags & 256;
        var allowEmpty = flags & 4096;
        flags = flags & ~6400;
        assert(!flags, `unknown flags in __syscall_newfstatat: ${flags}`);
        path = SYSCALLS.calculateAt(dirfd, path, allowEmpty);
        return SYSCALLS.doStat(nofollow ? FS.lstat : FS.stat, path, buf);
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    function ___syscall_openat(dirfd, path, flags, varargs) {
      SYSCALLS.varargs = varargs;
      try {
        path = SYSCALLS.getStr(path);
        path = SYSCALLS.calculateAt(dirfd, path);
        var mode = varargs ? syscallGetVarargI() : 0;
        return FS.open(path, flags, mode).fd;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    function ___syscall_rmdir(path) {
      try {
        path = SYSCALLS.getStr(path);
        FS.rmdir(path);
        return 0;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    function ___syscall_stat64(path, buf) {
      try {
        path = SYSCALLS.getStr(path);
        return SYSCALLS.doStat(FS.stat, path, buf);
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    function ___syscall_unlinkat(dirfd, path, flags) {
      try {
        path = SYSCALLS.getStr(path);
        path = SYSCALLS.calculateAt(dirfd, path);
        if (flags === 0) {
          FS.unlink(path);
        } else if (flags === 512) {
          FS.rmdir(path);
        } else {
          abort('Invalid flags passed to unlinkat');
        }
        return 0;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return -e.errno;
      }
    }

    var __abort_js = () => {
      abort('native code called abort()');
    };

    var __emscripten_memcpy_js = (dest, src, num) => HEAPU8.copyWithin(dest, src, src + num);

    var __emscripten_throw_longjmp = () => {
      throw Infinity;
    };

    function __gmtime_js(time_low, time_high, tmPtr) {
      var time = convertI32PairToI53Checked(time_low, time_high);

      var date = new Date(time * 1000);
      HEAP32[tmPtr >> 2] = date.getUTCSeconds();
      HEAP32[(tmPtr + 4) >> 2] = date.getUTCMinutes();
      HEAP32[(tmPtr + 8) >> 2] = date.getUTCHours();
      HEAP32[(tmPtr + 12) >> 2] = date.getUTCDate();
      HEAP32[(tmPtr + 16) >> 2] = date.getUTCMonth();
      HEAP32[(tmPtr + 20) >> 2] = date.getUTCFullYear() - 1900;
      HEAP32[(tmPtr + 24) >> 2] = date.getUTCDay();
      var start = Date.UTC(date.getUTCFullYear(), 0, 1, 0, 0, 0, 0);
      var yday = ((date.getTime() - start) / (1000 * 60 * 60 * 24)) | 0;
      HEAP32[(tmPtr + 28) >> 2] = yday;
    }

    var isLeapYear = (year) => year % 4 === 0 && (year % 100 !== 0 || year % 400 === 0);

    var MONTH_DAYS_LEAP_CUMULATIVE = [0, 31, 60, 91, 121, 152, 182, 213, 244, 274, 305, 335];

    var MONTH_DAYS_REGULAR_CUMULATIVE = [0, 31, 59, 90, 120, 151, 181, 212, 243, 273, 304, 334];
    var ydayFromDate = (date) => {
      var leap = isLeapYear(date.getFullYear());
      var monthDaysCumulative = leap ? MONTH_DAYS_LEAP_CUMULATIVE : MONTH_DAYS_REGULAR_CUMULATIVE;
      var yday = monthDaysCumulative[date.getMonth()] + date.getDate() - 1; // -1 since it's days since Jan 1

      return yday;
    };

    function __localtime_js(time_low, time_high, tmPtr) {
      var time = convertI32PairToI53Checked(time_low, time_high);

      var date = new Date(time * 1000);
      HEAP32[tmPtr >> 2] = date.getSeconds();
      HEAP32[(tmPtr + 4) >> 2] = date.getMinutes();
      HEAP32[(tmPtr + 8) >> 2] = date.getHours();
      HEAP32[(tmPtr + 12) >> 2] = date.getDate();
      HEAP32[(tmPtr + 16) >> 2] = date.getMonth();
      HEAP32[(tmPtr + 20) >> 2] = date.getFullYear() - 1900;
      HEAP32[(tmPtr + 24) >> 2] = date.getDay();

      var yday = ydayFromDate(date) | 0;
      HEAP32[(tmPtr + 28) >> 2] = yday;
      HEAP32[(tmPtr + 36) >> 2] = -(date.getTimezoneOffset() * 60);

      // Attention: DST is in December in South, and some regions don't have DST at all.
      var start = new Date(date.getFullYear(), 0, 1);
      var summerOffset = new Date(date.getFullYear(), 6, 1).getTimezoneOffset();
      var winterOffset = start.getTimezoneOffset();
      var dst =
        (summerOffset != winterOffset &&
          date.getTimezoneOffset() == Math.min(winterOffset, summerOffset)) | 0;
      HEAP32[(tmPtr + 32) >> 2] = dst;
    }

    var __tzset_js = (timezone, daylight, std_name, dst_name) => {
      // TODO: Use (malleable) environment variables instead of system settings.
      var currentYear = new Date().getFullYear();
      var winter = new Date(currentYear, 0, 1);
      var summer = new Date(currentYear, 6, 1);
      var winterOffset = winter.getTimezoneOffset();
      var summerOffset = summer.getTimezoneOffset();

      // Local standard timezone offset. Local standard time is not adjusted for
      // daylight savings.  This code uses the fact that getTimezoneOffset returns
      // a greater value during Standard Time versus Daylight Saving Time (DST).
      // Thus it determines the expected output during Standard Time, and it
      // compares whether the output of the given date the same (Standard) or less
      // (DST).
      var stdTimezoneOffset = Math.max(winterOffset, summerOffset);

      // timezone is specified as seconds west of UTC ("The external variable
      // `timezone` shall be set to the difference, in seconds, between
      // Coordinated Universal Time (UTC) and local standard time."), the same
      // as returned by stdTimezoneOffset.
      // See http://pubs.opengroup.org/onlinepubs/009695399/functions/tzset.html
      HEAPU32[timezone >> 2] = stdTimezoneOffset * 60;

      HEAP32[daylight >> 2] = Number(winterOffset != summerOffset);

      var extractZone = (timezoneOffset) => {
        // Why inverse sign?
        // Read here https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/getTimezoneOffset
        var sign = timezoneOffset >= 0 ? '-' : '+';

        var absOffset = Math.abs(timezoneOffset);
        var hours = String(Math.floor(absOffset / 60)).padStart(2, '0');
        var minutes = String(absOffset % 60).padStart(2, '0');

        return `UTC${sign}${hours}${minutes}`;
      };

      var winterName = extractZone(winterOffset);
      var summerName = extractZone(summerOffset);
      assert(winterName);
      assert(summerName);
      assert(
        lengthBytesUTF8(winterName) <= 16,
        `timezone name truncated to fit in TZNAME_MAX (${winterName})`,
      );
      assert(
        lengthBytesUTF8(summerName) <= 16,
        `timezone name truncated to fit in TZNAME_MAX (${summerName})`,
      );
      if (summerOffset < winterOffset) {
        // Northern hemisphere
        stringToUTF8(winterName, std_name, 17);
        stringToUTF8(summerName, dst_name, 17);
      } else {
        stringToUTF8(winterName, dst_name, 17);
        stringToUTF8(summerName, std_name, 17);
      }
    };

    var _emscripten_date_now = () => Date.now();

    var getHeapMax = () =>
      // Stay one Wasm page short of 4GB: while e.g. Chrome is able to allocate
      // full 4GB Wasm memories, the size will wrap back to 0 bytes in Wasm side
      // for any code that deals with heap sizes, which would require special
      // casing all heap size related code to treat 0 specially.
      2147483648;

    var growMemory = (size) => {
      var b = wasmMemory.buffer;
      var pages = ((size - b.byteLength + 65535) / 65536) | 0;
      try {
        // round size grow request up to wasm page size (fixed 64KB per spec)
        wasmMemory.grow(pages); // .grow() takes a delta compared to the previous size
        updateMemoryViews();
        return 1 /*success*/;
      } catch (e) {
        err(
          `growMemory: Attempted to grow heap from ${b.byteLength} bytes to ${size} bytes, but got error: ${e}`,
        );
      }
      // implicit 0 return to save code size (caller will cast "undefined" into 0
      // anyhow)
    };
    var _emscripten_resize_heap = (requestedSize) => {
      var oldSize = HEAPU8.length;
      // With CAN_ADDRESS_2GB or MEMORY64, pointers are already unsigned.
      requestedSize >>>= 0;
      // With multithreaded builds, races can happen (another thread might increase the size
      // in between), so return a failure, and let the caller retry.
      assert(requestedSize > oldSize);

      // Memory resize rules:
      // 1.  Always increase heap size to at least the requested size, rounded up
      //     to next page multiple.
      // 2a. If MEMORY_GROWTH_LINEAR_STEP == -1, excessively resize the heap
      //     geometrically: increase the heap size according to
      //     MEMORY_GROWTH_GEOMETRIC_STEP factor (default +20%), At most
      //     overreserve by MEMORY_GROWTH_GEOMETRIC_CAP bytes (default 96MB).
      // 2b. If MEMORY_GROWTH_LINEAR_STEP != -1, excessively resize the heap
      //     linearly: increase the heap size by at least
      //     MEMORY_GROWTH_LINEAR_STEP bytes.
      // 3.  Max size for the heap is capped at 2048MB-WASM_PAGE_SIZE, or by
      //     MAXIMUM_MEMORY, or by ASAN limit, depending on which is smallest
      // 4.  If we were unable to allocate as much memory, it may be due to
      //     over-eager decision to excessively reserve due to (3) above.
      //     Hence if an allocation fails, cut down on the amount of excess
      //     growth, in an attempt to succeed to perform a smaller allocation.

      // A limit is set for how much we can grow. We should not exceed that
      // (the wasm binary specifies it, so if we tried, we'd fail anyhow).
      var maxHeapSize = getHeapMax();
      if (requestedSize > maxHeapSize) {
        err(
          `Cannot enlarge memory, requested ${requestedSize} bytes, but the limit is ${maxHeapSize} bytes!`,
        );
        return false;
      }

      // Loop through potential heap size increases. If we attempt a too eager
      // reservation that fails, cut down on the attempted size and reserve a
      // smaller bump instead. (max 3 times, chosen somewhat arbitrarily)
      for (var cutDown = 1; cutDown <= 4; cutDown *= 2) {
        var overGrownHeapSize = oldSize * (1 + 0.2 / cutDown); // ensure geometric growth
        // but limit overreserving (default to capping at +96MB overgrowth at most)
        overGrownHeapSize = Math.min(overGrownHeapSize, requestedSize + 100663296);

        var newSize = Math.min(
          maxHeapSize,
          alignMemory(Math.max(requestedSize, overGrownHeapSize), 65536),
        );

        var replacement = growMemory(newSize);
        if (replacement) {
          return true;
        }
      }
      err(`Failed to grow the heap from ${oldSize} bytes to ${newSize} bytes, not enough memory!`);
      return false;
    };

    var ENV = {};

    var getExecutableName = () => {
      return thisProgram || './this.program';
    };
    var getEnvStrings = () => {
      if (!getEnvStrings.strings) {
        // Default values.
        // Browser language detection #8751
        var lang =
          (
            (typeof navigator == 'object' && navigator.languages && navigator.languages[0]) ||
            'C'
          ).replace('-', '_') + '.UTF-8';
        var env = {
          USER: 'web_user',
          LOGNAME: 'web_user',
          PATH: '/',
          PWD: '/',
          HOME: '/home/web_user',
          LANG: lang,
          _: getExecutableName(),
        };
        // Apply the user-provided values, if any.
        for (var x in ENV) {
          // x is a key in ENV; if ENV[x] is undefined, that means it was
          // explicitly set to be so. We allow user code to do that to
          // force variables with default values to remain unset.
          if (ENV[x] === undefined) delete env[x];
          else env[x] = ENV[x];
        }
        var strings = [];
        for (var x in env) {
          strings.push(`${x}=${env[x]}`);
        }
        getEnvStrings.strings = strings;
      }
      return getEnvStrings.strings;
    };

    var stringToAscii = (str, buffer) => {
      for (var i = 0; i < str.length; ++i) {
        assert(str.charCodeAt(i) === (str.charCodeAt(i) & 0xff));
        HEAP8[buffer++] = str.charCodeAt(i);
      }
      // Null-terminate the string
      HEAP8[buffer] = 0;
    };
    var _environ_get = (__environ, environ_buf) => {
      var bufSize = 0;
      getEnvStrings().forEach((string, i) => {
        var ptr = environ_buf + bufSize;
        HEAPU32[(__environ + i * 4) >> 2] = ptr;
        stringToAscii(string, ptr);
        bufSize += string.length + 1;
      });
      return 0;
    };

    var _environ_sizes_get = (penviron_count, penviron_buf_size) => {
      var strings = getEnvStrings();
      HEAPU32[penviron_count >> 2] = strings.length;
      var bufSize = 0;
      strings.forEach((string) => (bufSize += string.length + 1));
      HEAPU32[penviron_buf_size >> 2] = bufSize;
      return 0;
    };

    function _fd_close(fd) {
      try {
        var stream = SYSCALLS.getStreamFromFD(fd);
        FS.close(stream);
        return 0;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return e.errno;
      }
    }

    /** @param {number=} offset */
    var doReadv = (stream, iov, iovcnt, offset) => {
      var ret = 0;
      for (var i = 0; i < iovcnt; i++) {
        var ptr = HEAPU32[iov >> 2];
        var len = HEAPU32[(iov + 4) >> 2];
        iov += 8;
        var curr = FS.read(stream, HEAP8, ptr, len, offset);
        if (curr < 0) return -1;
        ret += curr;
        if (curr < len) break; // nothing more to read
        if (typeof offset != 'undefined') {
          offset += curr;
        }
      }
      return ret;
    };

    function _fd_read(fd, iov, iovcnt, pnum) {
      try {
        var stream = SYSCALLS.getStreamFromFD(fd);
        var num = doReadv(stream, iov, iovcnt);
        HEAPU32[pnum >> 2] = num;
        return 0;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return e.errno;
      }
    }

    function _fd_seek(fd, offset_low, offset_high, whence, newOffset) {
      var offset = convertI32PairToI53Checked(offset_low, offset_high);

      try {
        if (isNaN(offset)) return 61;
        var stream = SYSCALLS.getStreamFromFD(fd);
        FS.llseek(stream, offset, whence);
        (tempI64 = [
          stream.position >>> 0,
          ((tempDouble = stream.position),
          +Math.abs(tempDouble) >= 1.0
            ? tempDouble > 0.0
              ? +Math.floor(tempDouble / 4294967296.0) >>> 0
              : ~~+Math.ceil((tempDouble - +(~~tempDouble >>> 0)) / 4294967296.0) >>> 0
            : 0),
        ]),
          (HEAP32[newOffset >> 2] = tempI64[0]),
          (HEAP32[(newOffset + 4) >> 2] = tempI64[1]);
        if (stream.getdents && offset === 0 && whence === 0) stream.getdents = null; // reset readdir state
        return 0;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return e.errno;
      }
    }

    function _fd_sync(fd) {
      try {
        var stream = SYSCALLS.getStreamFromFD(fd);
        if (stream.stream_ops?.fsync) {
          return stream.stream_ops.fsync(stream);
        }
        return 0; // we can't do anything synchronously; the in-memory FS is already synced to
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return e.errno;
      }
    }

    /** @param {number=} offset */
    var doWritev = (stream, iov, iovcnt, offset) => {
      var ret = 0;
      for (var i = 0; i < iovcnt; i++) {
        var ptr = HEAPU32[iov >> 2];
        var len = HEAPU32[(iov + 4) >> 2];
        iov += 8;
        var curr = FS.write(stream, HEAP8, ptr, len, offset);
        if (curr < 0) return -1;
        ret += curr;
        if (curr < len) {
          // No more space to write.
          break;
        }
        if (typeof offset != 'undefined') {
          offset += curr;
        }
      }
      return ret;
    };

    function _fd_write(fd, iov, iovcnt, pnum) {
      try {
        var stream = SYSCALLS.getStreamFromFD(fd);
        var num = doWritev(stream, iov, iovcnt);
        HEAPU32[pnum >> 2] = num;
        return 0;
      } catch (e) {
        if (typeof FS == 'undefined' || !(e.name === 'ErrnoError')) throw e;
        return e.errno;
      }
    }

    var wasmTableMirror = [];

    /** @type {WebAssembly.Table} */
    var wasmTable;
    var getWasmTableEntry = (funcPtr) => {
      var func = wasmTableMirror[funcPtr];
      if (!func) {
        if (funcPtr >= wasmTableMirror.length) wasmTableMirror.length = funcPtr + 1;
        wasmTableMirror[funcPtr] = func = wasmTable.get(funcPtr);
      }
      assert(
        wasmTable.get(funcPtr) == func,
        'JavaScript-side Wasm function table mirror is out of date!',
      );
      return func;
    };

    var UTF16Decoder = typeof TextDecoder != 'undefined' ? new TextDecoder('utf-16le') : undefined;
    var UTF16ToString = (ptr, maxBytesToRead) => {
      assert(ptr % 2 == 0, 'Pointer passed to UTF16ToString must be aligned to two bytes!');
      var endPtr = ptr;
      // TextDecoder needs to know the byte length in advance, it doesn't stop on
      // null terminator by itself.
      // Also, use the length info to avoid running tiny strings through
      // TextDecoder, since .subarray() allocates garbage.
      var idx = endPtr >> 1;
      var maxIdx = idx + maxBytesToRead / 2;
      // If maxBytesToRead is not passed explicitly, it will be undefined, and this
      // will always evaluate to true. This saves on code size.
      while (!(idx >= maxIdx) && HEAPU16[idx]) ++idx;
      endPtr = idx << 1;

      if (endPtr - ptr > 32 && UTF16Decoder)
        return UTF16Decoder.decode(HEAPU8.subarray(ptr, endPtr));

      // Fallback: decode without UTF16Decoder
      var str = '';

      // If maxBytesToRead is not passed explicitly, it will be undefined, and the
      // for-loop's condition will always evaluate to true. The loop is then
      // terminated on the first null char.
      for (var i = 0; !(i >= maxBytesToRead / 2); ++i) {
        var codeUnit = HEAP16[(ptr + i * 2) >> 1];
        if (codeUnit == 0) break;
        // fromCharCode constructs a character from a UTF-16 code unit, so we can
        // pass the UTF16 string right through.
        str += String.fromCharCode(codeUnit);
      }

      return str;
    };

    var uleb128Encode = (n, target) => {
      assert(n < 16384);
      if (n < 128) {
        target.push(n);
      } else {
        target.push(n % 128 | 128, n >> 7);
      }
    };

    var sigToWasmTypes = (sig) => {
      assert(
        !sig.includes('j'),
        'i64 not permitted in function signatures when WASM_BIGINT is disabled',
      );
      var typeNames = {
        i: 'i32',
        j: 'i64',
        f: 'f32',
        d: 'f64',
        e: 'externref',
        p: 'i32',
      };
      var type = {
        parameters: [],
        results: sig[0] == 'v' ? [] : [typeNames[sig[0]]],
      };
      for (var i = 1; i < sig.length; ++i) {
        assert(sig[i] in typeNames, 'invalid signature char: ' + sig[i]);
        type.parameters.push(typeNames[sig[i]]);
      }
      return type;
    };

    var generateFuncType = (sig, target) => {
      var sigRet = sig.slice(0, 1);
      var sigParam = sig.slice(1);
      var typeCodes = {
        i: 0x7f, // i32
        p: 0x7f, // i32
        j: 0x7e, // i64
        f: 0x7d, // f32
        d: 0x7c, // f64
        e: 0x6f, // externref
      };

      // Parameters, length + signatures
      target.push(0x60 /* form: func */);
      uleb128Encode(sigParam.length, target);
      for (var i = 0; i < sigParam.length; ++i) {
        assert(sigParam[i] in typeCodes, 'invalid signature char: ' + sigParam[i]);
        target.push(typeCodes[sigParam[i]]);
      }

      // Return values, length + signatures
      // With no multi-return in MVP, either 0 (void) or 1 (anything else)
      if (sigRet == 'v') {
        target.push(0x00);
      } else {
        target.push(0x01, typeCodes[sigRet]);
      }
    };
    var convertJsFunctionToWasm = (func, sig) => {
      assert(
        !sig.includes('j'),
        'i64 not permitted in function signatures when WASM_BIGINT is disabled',
      );

      // If the type reflection proposal is available, use the new
      // "WebAssembly.Function" constructor.
      // Otherwise, construct a minimal wasm module importing the JS function and
      // re-exporting it.
      if (typeof WebAssembly.Function == 'function') {
        return new WebAssembly.Function(sigToWasmTypes(sig), func);
      }

      // The module is static, with the exception of the type section, which is
      // generated based on the signature passed in.
      var typeSectionBody = [
        0x01, // count: 1
      ];
      generateFuncType(sig, typeSectionBody);

      // Rest of the module is static
      var bytes = [
        0x00,
        0x61,
        0x73,
        0x6d, // magic ("\0asm")
        0x01,
        0x00,
        0x00,
        0x00, // version: 1
        0x01, // Type section code
      ];
      // Write the overall length of the type section followed by the body
      uleb128Encode(typeSectionBody.length, bytes);
      bytes.push(...typeSectionBody);

      // The rest of the module is static
      bytes.push(
        0x02,
        0x07, // import section
        // (import "e" "f" (func 0 (type 0)))
        0x01,
        0x01,
        0x65,
        0x01,
        0x66,
        0x00,
        0x00,
        0x07,
        0x05, // export section
        // (export "f" (func 0 (type 0)))
        0x01,
        0x01,
        0x66,
        0x00,
        0x00,
      );

      // We can compile this wasm module synchronously because it is very small.
      // This accepts an import (at "e.f"), that it reroutes to an export (at "f")
      var module = new WebAssembly.Module(new Uint8Array(bytes));
      var instance = new WebAssembly.Instance(module, { e: { f: func } });
      var wrappedFunc = instance.exports['f'];
      return wrappedFunc;
    };

    var updateTableMap = (offset, count) => {
      if (functionsInTableMap) {
        for (var i = offset; i < offset + count; i++) {
          var item = getWasmTableEntry(i);
          // Ignore null values.
          if (item) {
            functionsInTableMap.set(item, i);
          }
        }
      }
    };

    var functionsInTableMap;

    var getFunctionAddress = (func) => {
      // First, create the map if this is the first use.
      if (!functionsInTableMap) {
        functionsInTableMap = new WeakMap();
        updateTableMap(0, wasmTable.length);
      }
      return functionsInTableMap.get(func) || 0;
    };

    var freeTableIndexes = [];

    var getEmptyTableSlot = () => {
      // Reuse a free index if there is one, otherwise grow.
      if (freeTableIndexes.length) {
        return freeTableIndexes.pop();
      }
      // Grow the table
      try {
        wasmTable.grow(1);
      } catch (err) {
        if (!(err instanceof RangeError)) {
          throw err;
        }
        throw 'Unable to grow wasm table. Set ALLOW_TABLE_GROWTH.';
      }
      return wasmTable.length - 1;
    };

    var setWasmTableEntry = (idx, func) => {
      wasmTable.set(idx, func);
      // With ABORT_ON_WASM_EXCEPTIONS wasmTable.get is overridden to return wrapped
      // functions so we need to call it here to retrieve the potential wrapper correctly
      // instead of just storing 'func' directly into wasmTableMirror
      wasmTableMirror[idx] = wasmTable.get(idx);
    };

    /** @param {string=} sig */
    var addFunction = (func, sig) => {
      assert(typeof func != 'undefined');
      // Check if the function is already in the table, to ensure each function
      // gets a unique index.
      var rtn = getFunctionAddress(func);
      if (rtn) {
        return rtn;
      }

      // It's not in the table, add it now.

      var ret = getEmptyTableSlot();

      // Set the new value.
      try {
        // Attempting to call this with JS function will cause of table.set() to fail
        setWasmTableEntry(ret, func);
      } catch (err) {
        if (!(err instanceof TypeError)) {
          throw err;
        }
        assert(typeof sig != 'undefined', 'Missing signature argument to addFunction: ' + func);
        var wrapped = convertJsFunctionToWasm(func, sig);
        setWasmTableEntry(ret, wrapped);
      }

      functionsInTableMap.set(func, ret);

      return ret;
    };

    var getCFunc = (ident) => {
      var func = Module['_' + ident]; // closure exported function
      assert(func, 'Cannot call unknown function ' + ident + ', make sure it is exported');
      return func;
    };

    var writeArrayToMemory = (array, buffer) => {
      assert(
        array.length >= 0,
        'writeArrayToMemory array must have a length (should be an array or typed array)',
      );
      HEAP8.set(array, buffer);
    };

    var stackAlloc = (sz) => __emscripten_stack_alloc(sz);
    var stringToUTF8OnStack = (str) => {
      var size = lengthBytesUTF8(str) + 1;
      var ret = stackAlloc(size);
      stringToUTF8(str, ret, size);
      return ret;
    };

    /**
     * @param {string|null=} returnType
     * @param {Array=} argTypes
     * @param {Arguments|Array=} args
     * @param {Object=} opts
     */
    var ccall = (ident, returnType, argTypes, args, opts) => {
      // For fast lookup of conversion functions
      var toC = {
        string: (str) => {
          var ret = 0;
          if (str !== null && str !== undefined && str !== 0) {
            // null string
            ret = stringToUTF8OnStack(str);
          }
          return ret;
        },
        array: (arr) => {
          var ret = stackAlloc(arr.length);
          writeArrayToMemory(arr, ret);
          return ret;
        },
      };

      function convertReturnValue(ret) {
        if (returnType === 'string') {
          return UTF8ToString(ret);
        }
        if (returnType === 'boolean') return Boolean(ret);
        return ret;
      }

      var func = getCFunc(ident);
      var cArgs = [];
      var stack = 0;
      assert(returnType !== 'array', 'Return type should not be "array".');
      if (args) {
        for (var i = 0; i < args.length; i++) {
          var converter = toC[argTypes[i]];
          if (converter) {
            if (stack === 0) stack = stackSave();
            cArgs[i] = converter(args[i]);
          } else {
            cArgs[i] = args[i];
          }
        }
      }
      var ret = func(...cArgs);
      function onDone(ret) {
        if (stack !== 0) stackRestore(stack);
        return convertReturnValue(ret);
      }

      ret = onDone(ret);
      return ret;
    };

    /**
     * @param {string=} returnType
     * @param {Array=} argTypes
     * @param {Object=} opts
     */
    var cwrap = (ident, returnType, argTypes, opts) => {
      return (...args) => ccall(ident, returnType, argTypes, args, opts);
    };

    var removeFunction = (index) => {
      functionsInTableMap.delete(getWasmTableEntry(index));
      setWasmTableEntry(index, null);
      freeTableIndexes.push(index);
    };

    var stringToUTF16 = (str, outPtr, maxBytesToWrite) => {
      assert(outPtr % 2 == 0, 'Pointer passed to stringToUTF16 must be aligned to two bytes!');
      assert(
        typeof maxBytesToWrite == 'number',
        'stringToUTF16(str, outPtr, maxBytesToWrite) is missing the third parameter that specifies the length of the output buffer!',
      );
      // Backwards compatibility: if max bytes is not specified, assume unsafe unbounded write is allowed.
      maxBytesToWrite ??= 0x7fffffff;
      if (maxBytesToWrite < 2) return 0;
      maxBytesToWrite -= 2; // Null terminator.
      var startPtr = outPtr;
      var numCharsToWrite = maxBytesToWrite < str.length * 2 ? maxBytesToWrite / 2 : str.length;
      for (var i = 0; i < numCharsToWrite; ++i) {
        // charCodeAt returns a UTF-16 encoded code unit, so it can be directly written to the HEAP.
        var codeUnit = str.charCodeAt(i); // possibly a lead surrogate
        HEAP16[outPtr >> 1] = codeUnit;
        outPtr += 2;
      }
      // Null-terminate the pointer to the HEAP.
      HEAP16[outPtr >> 1] = 0;
      return outPtr - startPtr;
    };

    FS.createPreloadedFile = FS_createPreloadedFile;
    FS.staticInit();
    // Set module methods based on EXPORTED_RUNTIME_METHODS
    function checkIncomingModuleAPI() {
      ignoredModuleProp('fetchSettings');
    }
    var wasmImports = {
      /** @export */
      __assert_fail: ___assert_fail,
      /** @export */
      __syscall_fcntl64: ___syscall_fcntl64,
      /** @export */
      __syscall_fstat64: ___syscall_fstat64,
      /** @export */
      __syscall_ftruncate64: ___syscall_ftruncate64,
      /** @export */
      __syscall_getdents64: ___syscall_getdents64,
      /** @export */
      __syscall_ioctl: ___syscall_ioctl,
      /** @export */
      __syscall_lstat64: ___syscall_lstat64,
      /** @export */
      __syscall_newfstatat: ___syscall_newfstatat,
      /** @export */
      __syscall_openat: ___syscall_openat,
      /** @export */
      __syscall_rmdir: ___syscall_rmdir,
      /** @export */
      __syscall_stat64: ___syscall_stat64,
      /** @export */
      __syscall_unlinkat: ___syscall_unlinkat,
      /** @export */
      _abort_js: __abort_js,
      /** @export */
      _emscripten_memcpy_js: __emscripten_memcpy_js,
      /** @export */
      _emscripten_throw_longjmp: __emscripten_throw_longjmp,
      /** @export */
      _gmtime_js: __gmtime_js,
      /** @export */
      _localtime_js: __localtime_js,
      /** @export */
      _tzset_js: __tzset_js,
      /** @export */
      emscripten_date_now: _emscripten_date_now,
      /** @export */
      emscripten_resize_heap: _emscripten_resize_heap,
      /** @export */
      environ_get: _environ_get,
      /** @export */
      environ_sizes_get: _environ_sizes_get,
      /** @export */
      fd_close: _fd_close,
      /** @export */
      fd_read: _fd_read,
      /** @export */
      fd_seek: _fd_seek,
      /** @export */
      fd_sync: _fd_sync,
      /** @export */
      fd_write: _fd_write,
      /** @export */
      invoke_ii,
      /** @export */
      invoke_iii,
      /** @export */
      invoke_iiii,
      /** @export */
      invoke_iiiii,
      /** @export */
      invoke_v,
      /** @export */
      invoke_vii,
      /** @export */
      invoke_viii,
      /** @export */
      invoke_viiii,
      /** @export */
      invoke_viiiiiiiii,
    };
    var wasmExports = createWasm();
    var ___wasm_call_ctors = createExportWrapper('__wasm_call_ctors', 0);
    var _PDFiumExt_Init = (Module['_PDFiumExt_Init'] = createExportWrapper('PDFiumExt_Init', 0));
    var _FPDF_InitLibraryWithConfig = (Module['_FPDF_InitLibraryWithConfig'] = createExportWrapper(
      'FPDF_InitLibraryWithConfig',
      1,
    ));
    var _PDFiumExt_OpenFileWriter = (Module['_PDFiumExt_OpenFileWriter'] = createExportWrapper(
      'PDFiumExt_OpenFileWriter',
      0,
    ));
    var _PDFiumExt_GetFileWriterSize = (Module['_PDFiumExt_GetFileWriterSize'] =
      createExportWrapper('PDFiumExt_GetFileWriterSize', 1));
    var _PDFiumExt_GetFileWriterData = (Module['_PDFiumExt_GetFileWriterData'] =
      createExportWrapper('PDFiumExt_GetFileWriterData', 3));
    var _PDFiumExt_CloseFileWriter = (Module['_PDFiumExt_CloseFileWriter'] = createExportWrapper(
      'PDFiumExt_CloseFileWriter',
      1,
    ));
    var _PDFiumExt_SaveAsCopy = (Module['_PDFiumExt_SaveAsCopy'] = createExportWrapper(
      'PDFiumExt_SaveAsCopy',
      2,
    ));
    var _FPDF_SaveAsCopy = (Module['_FPDF_SaveAsCopy'] = createExportWrapper('FPDF_SaveAsCopy', 3));
    var _PDFiumExt_OpenFormFillInfo = (Module['_PDFiumExt_OpenFormFillInfo'] = createExportWrapper(
      'PDFiumExt_OpenFormFillInfo',
      0,
    ));
    var _PDFiumExt_CloseFormFillInfo = (Module['_PDFiumExt_CloseFormFillInfo'] =
      createExportWrapper('PDFiumExt_CloseFormFillInfo', 1));
    var _PDFiumExt_InitFormFillEnvironment = (Module['_PDFiumExt_InitFormFillEnvironment'] =
      createExportWrapper('PDFiumExt_InitFormFillEnvironment', 2));
    var _FPDFDOC_InitFormFillEnvironment = (Module['_FPDFDOC_InitFormFillEnvironment'] =
      createExportWrapper('FPDFDOC_InitFormFillEnvironment', 2));
    var _PDFiumExt_ExitFormFillEnvironment = (Module['_PDFiumExt_ExitFormFillEnvironment'] =
      createExportWrapper('PDFiumExt_ExitFormFillEnvironment', 1));
    var _FPDFDOC_ExitFormFillEnvironment = (Module['_FPDFDOC_ExitFormFillEnvironment'] =
      createExportWrapper('FPDFDOC_ExitFormFillEnvironment', 1));
    var _EPDFNamedDest_SetDest = (Module['_EPDFNamedDest_SetDest'] = createExportWrapper(
      'EPDFNamedDest_SetDest',
      3,
    ));
    var _EPDFNamedDest_Remove = (Module['_EPDFNamedDest_Remove'] = createExportWrapper(
      'EPDFNamedDest_Remove',
      2,
    ));
    var _EPDFDest_CreateView = (Module['_EPDFDest_CreateView'] = createExportWrapper(
      'EPDFDest_CreateView',
      4,
    ));
    var _EPDFDest_CreateXYZ = (Module['_EPDFDest_CreateXYZ'] = createExportWrapper(
      'EPDFDest_CreateXYZ',
      7,
    ));
    var _EPDFDest_CreateRemoteView = (Module['_EPDFDest_CreateRemoteView'] = createExportWrapper(
      'EPDFDest_CreateRemoteView',
      5,
    ));
    var _EPDFDest_CreateRemoteXYZ = (Module['_EPDFDest_CreateRemoteXYZ'] = createExportWrapper(
      'EPDFDest_CreateRemoteXYZ',
      8,
    ));
    var _EPDFAction_CreateGoTo = (Module['_EPDFAction_CreateGoTo'] = createExportWrapper(
      'EPDFAction_CreateGoTo',
      2,
    ));
    var _EPDFAction_CreateGoToNamed = (Module['_EPDFAction_CreateGoToNamed'] = createExportWrapper(
      'EPDFAction_CreateGoToNamed',
      2,
    ));
    var _EPDFAction_CreateLaunch = (Module['_EPDFAction_CreateLaunch'] = createExportWrapper(
      'EPDFAction_CreateLaunch',
      2,
    ));
    var _EPDFAction_CreateRemoteGoToByName = (Module['_EPDFAction_CreateRemoteGoToByName'] =
      createExportWrapper('EPDFAction_CreateRemoteGoToByName', 3));
    var _EPDFAction_CreateRemoteGoToDest = (Module['_EPDFAction_CreateRemoteGoToDest'] =
      createExportWrapper('EPDFAction_CreateRemoteGoToDest', 3));
    var _EPDFAction_CreateURI = (Module['_EPDFAction_CreateURI'] = createExportWrapper(
      'EPDFAction_CreateURI',
      2,
    ));
    var _EPDFBookmark_Create = (Module['_EPDFBookmark_Create'] = createExportWrapper(
      'EPDFBookmark_Create',
      2,
    ));
    var _EPDFBookmark_Delete = (Module['_EPDFBookmark_Delete'] = createExportWrapper(
      'EPDFBookmark_Delete',
      2,
    ));
    var _EPDFBookmark_AppendChild = (Module['_EPDFBookmark_AppendChild'] = createExportWrapper(
      'EPDFBookmark_AppendChild',
      3,
    ));
    var _EPDFBookmark_InsertAfter = (Module['_EPDFBookmark_InsertAfter'] = createExportWrapper(
      'EPDFBookmark_InsertAfter',
      4,
    ));
    var _EPDFBookmark_Clear = (Module['_EPDFBookmark_Clear'] = createExportWrapper(
      'EPDFBookmark_Clear',
      1,
    ));
    var _EPDFBookmark_SetTitle = (Module['_EPDFBookmark_SetTitle'] = createExportWrapper(
      'EPDFBookmark_SetTitle',
      2,
    ));
    var _EPDFBookmark_SetDest = (Module['_EPDFBookmark_SetDest'] = createExportWrapper(
      'EPDFBookmark_SetDest',
      3,
    ));
    var _EPDFBookmark_SetAction = (Module['_EPDFBookmark_SetAction'] = createExportWrapper(
      'EPDFBookmark_SetAction',
      3,
    ));
    var _EPDFBookmark_ClearTarget = (Module['_EPDFBookmark_ClearTarget'] = createExportWrapper(
      'EPDFBookmark_ClearTarget',
      1,
    ));
    var _EPDF_PNG_EncodeRGBA = (Module['_EPDF_PNG_EncodeRGBA'] = createExportWrapper(
      'EPDF_PNG_EncodeRGBA',
      6,
    ));
    var _FPDFAnnot_IsSupportedSubtype = (Module['_FPDFAnnot_IsSupportedSubtype'] =
      createExportWrapper('FPDFAnnot_IsSupportedSubtype', 1));
    var _FPDFPage_CreateAnnot = (Module['_FPDFPage_CreateAnnot'] = createExportWrapper(
      'FPDFPage_CreateAnnot',
      2,
    ));
    var _FPDFPage_GetAnnotCount = (Module['_FPDFPage_GetAnnotCount'] = createExportWrapper(
      'FPDFPage_GetAnnotCount',
      1,
    ));
    var _FPDFPage_GetAnnot = (Module['_FPDFPage_GetAnnot'] = createExportWrapper(
      'FPDFPage_GetAnnot',
      2,
    ));
    var _FPDFPage_GetAnnotIndex = (Module['_FPDFPage_GetAnnotIndex'] = createExportWrapper(
      'FPDFPage_GetAnnotIndex',
      2,
    ));
    var _FPDFPage_CloseAnnot = (Module['_FPDFPage_CloseAnnot'] = createExportWrapper(
      'FPDFPage_CloseAnnot',
      1,
    ));
    var _FPDFPage_RemoveAnnot = (Module['_FPDFPage_RemoveAnnot'] = createExportWrapper(
      'FPDFPage_RemoveAnnot',
      2,
    ));
    var _FPDFAnnot_GetSubtype = (Module['_FPDFAnnot_GetSubtype'] = createExportWrapper(
      'FPDFAnnot_GetSubtype',
      1,
    ));
    var _FPDFAnnot_IsObjectSupportedSubtype = (Module['_FPDFAnnot_IsObjectSupportedSubtype'] =
      createExportWrapper('FPDFAnnot_IsObjectSupportedSubtype', 1));
    var _FPDFAnnot_UpdateObject = (Module['_FPDFAnnot_UpdateObject'] = createExportWrapper(
      'FPDFAnnot_UpdateObject',
      2,
    ));
    var _FPDFAnnot_AddInkStroke = (Module['_FPDFAnnot_AddInkStroke'] = createExportWrapper(
      'FPDFAnnot_AddInkStroke',
      3,
    ));
    var _FPDFAnnot_RemoveInkList = (Module['_FPDFAnnot_RemoveInkList'] = createExportWrapper(
      'FPDFAnnot_RemoveInkList',
      1,
    ));
    var _FPDFAnnot_AppendObject = (Module['_FPDFAnnot_AppendObject'] = createExportWrapper(
      'FPDFAnnot_AppendObject',
      2,
    ));
    var _FPDFAnnot_GetObjectCount = (Module['_FPDFAnnot_GetObjectCount'] = createExportWrapper(
      'FPDFAnnot_GetObjectCount',
      1,
    ));
    var _FPDFAnnot_GetObject = (Module['_FPDFAnnot_GetObject'] = createExportWrapper(
      'FPDFAnnot_GetObject',
      2,
    ));
    var _FPDFAnnot_RemoveObject = (Module['_FPDFAnnot_RemoveObject'] = createExportWrapper(
      'FPDFAnnot_RemoveObject',
      2,
    ));
    var _FPDFAnnot_SetColor = (Module['_FPDFAnnot_SetColor'] = createExportWrapper(
      'FPDFAnnot_SetColor',
      6,
    ));
    var _FPDFAnnot_GetColor = (Module['_FPDFAnnot_GetColor'] = createExportWrapper(
      'FPDFAnnot_GetColor',
      6,
    ));
    var _FPDFAnnot_HasAttachmentPoints = (Module['_FPDFAnnot_HasAttachmentPoints'] =
      createExportWrapper('FPDFAnnot_HasAttachmentPoints', 1));
    var _FPDFAnnot_SetAttachmentPoints = (Module['_FPDFAnnot_SetAttachmentPoints'] =
      createExportWrapper('FPDFAnnot_SetAttachmentPoints', 3));
    var _FPDFAnnot_AppendAttachmentPoints = (Module['_FPDFAnnot_AppendAttachmentPoints'] =
      createExportWrapper('FPDFAnnot_AppendAttachmentPoints', 2));
    var _FPDFAnnot_CountAttachmentPoints = (Module['_FPDFAnnot_CountAttachmentPoints'] =
      createExportWrapper('FPDFAnnot_CountAttachmentPoints', 1));
    var _FPDFAnnot_GetAttachmentPoints = (Module['_FPDFAnnot_GetAttachmentPoints'] =
      createExportWrapper('FPDFAnnot_GetAttachmentPoints', 3));
    var _FPDFAnnot_SetRect = (Module['_FPDFAnnot_SetRect'] = createExportWrapper(
      'FPDFAnnot_SetRect',
      2,
    ));
    var _FPDFAnnot_GetRect = (Module['_FPDFAnnot_GetRect'] = createExportWrapper(
      'FPDFAnnot_GetRect',
      2,
    ));
    var _FPDFAnnot_GetVertices = (Module['_FPDFAnnot_GetVertices'] = createExportWrapper(
      'FPDFAnnot_GetVertices',
      3,
    ));
    var _FPDFAnnot_GetInkListCount = (Module['_FPDFAnnot_GetInkListCount'] = createExportWrapper(
      'FPDFAnnot_GetInkListCount',
      1,
    ));
    var _FPDFAnnot_GetInkListPath = (Module['_FPDFAnnot_GetInkListPath'] = createExportWrapper(
      'FPDFAnnot_GetInkListPath',
      4,
    ));
    var _FPDFAnnot_GetLine = (Module['_FPDFAnnot_GetLine'] = createExportWrapper(
      'FPDFAnnot_GetLine',
      3,
    ));
    var _FPDFAnnot_SetBorder = (Module['_FPDFAnnot_SetBorder'] = createExportWrapper(
      'FPDFAnnot_SetBorder',
      4,
    ));
    var _FPDFAnnot_GetBorder = (Module['_FPDFAnnot_GetBorder'] = createExportWrapper(
      'FPDFAnnot_GetBorder',
      4,
    ));
    var _FPDFAnnot_HasKey = (Module['_FPDFAnnot_HasKey'] = createExportWrapper(
      'FPDFAnnot_HasKey',
      2,
    ));
    var _FPDFAnnot_GetValueType = (Module['_FPDFAnnot_GetValueType'] = createExportWrapper(
      'FPDFAnnot_GetValueType',
      2,
    ));
    var _FPDFAnnot_SetStringValue = (Module['_FPDFAnnot_SetStringValue'] = createExportWrapper(
      'FPDFAnnot_SetStringValue',
      3,
    ));
    var _FPDFAnnot_GetStringValue = (Module['_FPDFAnnot_GetStringValue'] = createExportWrapper(
      'FPDFAnnot_GetStringValue',
      4,
    ));
    var _FPDFAnnot_GetNumberValue = (Module['_FPDFAnnot_GetNumberValue'] = createExportWrapper(
      'FPDFAnnot_GetNumberValue',
      3,
    ));
    var _FPDFAnnot_SetAP = (Module['_FPDFAnnot_SetAP'] = createExportWrapper('FPDFAnnot_SetAP', 3));
    var _FPDFAnnot_GetAP = (Module['_FPDFAnnot_GetAP'] = createExportWrapper('FPDFAnnot_GetAP', 4));
    var _FPDFAnnot_GetLinkedAnnot = (Module['_FPDFAnnot_GetLinkedAnnot'] = createExportWrapper(
      'FPDFAnnot_GetLinkedAnnot',
      2,
    ));
    var _FPDFAnnot_GetFlags = (Module['_FPDFAnnot_GetFlags'] = createExportWrapper(
      'FPDFAnnot_GetFlags',
      1,
    ));
    var _FPDFAnnot_SetFlags = (Module['_FPDFAnnot_SetFlags'] = createExportWrapper(
      'FPDFAnnot_SetFlags',
      2,
    ));
    var _FPDFAnnot_GetFormFieldFlags = (Module['_FPDFAnnot_GetFormFieldFlags'] =
      createExportWrapper('FPDFAnnot_GetFormFieldFlags', 2));
    var _FPDFAnnot_SetFormFieldFlags = (Module['_FPDFAnnot_SetFormFieldFlags'] =
      createExportWrapper('FPDFAnnot_SetFormFieldFlags', 3));
    var _FPDFAnnot_GetFormFieldAtPoint = (Module['_FPDFAnnot_GetFormFieldAtPoint'] =
      createExportWrapper('FPDFAnnot_GetFormFieldAtPoint', 3));
    var _FPDFAnnot_GetFormFieldName = (Module['_FPDFAnnot_GetFormFieldName'] = createExportWrapper(
      'FPDFAnnot_GetFormFieldName',
      4,
    ));
    var _FPDFAnnot_GetFormFieldType = (Module['_FPDFAnnot_GetFormFieldType'] = createExportWrapper(
      'FPDFAnnot_GetFormFieldType',
      2,
    ));
    var _FPDFAnnot_GetFormAdditionalActionJavaScript = (Module[
      '_FPDFAnnot_GetFormAdditionalActionJavaScript'
    ] = createExportWrapper('FPDFAnnot_GetFormAdditionalActionJavaScript', 5));
    var _FPDFAnnot_GetFormFieldAlternateName = (Module['_FPDFAnnot_GetFormFieldAlternateName'] =
      createExportWrapper('FPDFAnnot_GetFormFieldAlternateName', 4));
    var _FPDFAnnot_GetFormFieldValue = (Module['_FPDFAnnot_GetFormFieldValue'] =
      createExportWrapper('FPDFAnnot_GetFormFieldValue', 4));
    var _FPDFAnnot_GetOptionCount = (Module['_FPDFAnnot_GetOptionCount'] = createExportWrapper(
      'FPDFAnnot_GetOptionCount',
      2,
    ));
    var _FPDFAnnot_GetOptionLabel = (Module['_FPDFAnnot_GetOptionLabel'] = createExportWrapper(
      'FPDFAnnot_GetOptionLabel',
      5,
    ));
    var _FPDFAnnot_IsOptionSelected = (Module['_FPDFAnnot_IsOptionSelected'] = createExportWrapper(
      'FPDFAnnot_IsOptionSelected',
      3,
    ));
    var _FPDFAnnot_GetFontSize = (Module['_FPDFAnnot_GetFontSize'] = createExportWrapper(
      'FPDFAnnot_GetFontSize',
      3,
    ));
    var _FPDFAnnot_SetFontColor = (Module['_FPDFAnnot_SetFontColor'] = createExportWrapper(
      'FPDFAnnot_SetFontColor',
      5,
    ));
    var _FPDFAnnot_GetFontColor = (Module['_FPDFAnnot_GetFontColor'] = createExportWrapper(
      'FPDFAnnot_GetFontColor',
      5,
    ));
    var _FPDFAnnot_IsChecked = (Module['_FPDFAnnot_IsChecked'] = createExportWrapper(
      'FPDFAnnot_IsChecked',
      2,
    ));
    var _FPDFAnnot_SetFocusableSubtypes = (Module['_FPDFAnnot_SetFocusableSubtypes'] =
      createExportWrapper('FPDFAnnot_SetFocusableSubtypes', 3));
    var _FPDFAnnot_GetFocusableSubtypesCount = (Module['_FPDFAnnot_GetFocusableSubtypesCount'] =
      createExportWrapper('FPDFAnnot_GetFocusableSubtypesCount', 1));
    var _FPDFAnnot_GetFocusableSubtypes = (Module['_FPDFAnnot_GetFocusableSubtypes'] =
      createExportWrapper('FPDFAnnot_GetFocusableSubtypes', 3));
    var _FPDFAnnot_GetLink = (Module['_FPDFAnnot_GetLink'] = createExportWrapper(
      'FPDFAnnot_GetLink',
      1,
    ));
    var _FPDFAnnot_GetFormControlCount = (Module['_FPDFAnnot_GetFormControlCount'] =
      createExportWrapper('FPDFAnnot_GetFormControlCount', 2));
    var _FPDFAnnot_GetFormControlIndex = (Module['_FPDFAnnot_GetFormControlIndex'] =
      createExportWrapper('FPDFAnnot_GetFormControlIndex', 2));
    var _FPDFAnnot_GetFormFieldExportValue = (Module['_FPDFAnnot_GetFormFieldExportValue'] =
      createExportWrapper('FPDFAnnot_GetFormFieldExportValue', 4));
    var _FPDFAnnot_SetURI = (Module['_FPDFAnnot_SetURI'] = createExportWrapper(
      'FPDFAnnot_SetURI',
      2,
    ));
    var _FPDFAnnot_GetFileAttachment = (Module['_FPDFAnnot_GetFileAttachment'] =
      createExportWrapper('FPDFAnnot_GetFileAttachment', 1));
    var _FPDFAnnot_AddFileAttachment = (Module['_FPDFAnnot_AddFileAttachment'] =
      createExportWrapper('FPDFAnnot_AddFileAttachment', 2));
    var _EPDFAnnot_SetColor = (Module['_EPDFAnnot_SetColor'] = createExportWrapper(
      'EPDFAnnot_SetColor',
      5,
    ));
    var _EPDFAnnot_GetColor = (Module['_EPDFAnnot_GetColor'] = createExportWrapper(
      'EPDFAnnot_GetColor',
      5,
    ));
    var _EPDFAnnot_ClearColor = (Module['_EPDFAnnot_ClearColor'] = createExportWrapper(
      'EPDFAnnot_ClearColor',
      2,
    ));
    var _EPDFAnnot_SetOpacity = (Module['_EPDFAnnot_SetOpacity'] = createExportWrapper(
      'EPDFAnnot_SetOpacity',
      2,
    ));
    var _EPDFAnnot_GetOpacity = (Module['_EPDFAnnot_GetOpacity'] = createExportWrapper(
      'EPDFAnnot_GetOpacity',
      2,
    ));
    var _EPDFAnnot_GetBorderEffect = (Module['_EPDFAnnot_GetBorderEffect'] = createExportWrapper(
      'EPDFAnnot_GetBorderEffect',
      2,
    ));
    var _EPDFAnnot_GetRectangleDifferences = (Module['_EPDFAnnot_GetRectangleDifferences'] =
      createExportWrapper('EPDFAnnot_GetRectangleDifferences', 5));
    var _EPDFAnnot_GetBorderDashPatternCount = (Module['_EPDFAnnot_GetBorderDashPatternCount'] =
      createExportWrapper('EPDFAnnot_GetBorderDashPatternCount', 1));
    var _EPDFAnnot_GetBorderDashPattern = (Module['_EPDFAnnot_GetBorderDashPattern'] =
      createExportWrapper('EPDFAnnot_GetBorderDashPattern', 3));
    var _EPDFAnnot_SetBorderDashPattern = (Module['_EPDFAnnot_SetBorderDashPattern'] =
      createExportWrapper('EPDFAnnot_SetBorderDashPattern', 3));
    var _EPDFAnnot_GetBorderStyle = (Module['_EPDFAnnot_GetBorderStyle'] = createExportWrapper(
      'EPDFAnnot_GetBorderStyle',
      2,
    ));
    var _EPDFAnnot_SetBorderStyle = (Module['_EPDFAnnot_SetBorderStyle'] = createExportWrapper(
      'EPDFAnnot_SetBorderStyle',
      3,
    ));
    var _EPDFAnnot_GenerateAppearance = (Module['_EPDFAnnot_GenerateAppearance'] =
      createExportWrapper('EPDFAnnot_GenerateAppearance', 1));
    var _EPDFAnnot_GenerateAppearanceWithBlend = (Module['_EPDFAnnot_GenerateAppearanceWithBlend'] =
      createExportWrapper('EPDFAnnot_GenerateAppearanceWithBlend', 2));
    var _EPDFAnnot_GetBlendMode = (Module['_EPDFAnnot_GetBlendMode'] = createExportWrapper(
      'EPDFAnnot_GetBlendMode',
      1,
    ));
    var _EPDFAnnot_SetIntent = (Module['_EPDFAnnot_SetIntent'] = createExportWrapper(
      'EPDFAnnot_SetIntent',
      2,
    ));
    var _EPDFAnnot_GetIntent = (Module['_EPDFAnnot_GetIntent'] = createExportWrapper(
      'EPDFAnnot_GetIntent',
      3,
    ));
    var _EPDFAnnot_GetRichContent = (Module['_EPDFAnnot_GetRichContent'] = createExportWrapper(
      'EPDFAnnot_GetRichContent',
      3,
    ));
    var _EPDFAnnot_SetLineEndings = (Module['_EPDFAnnot_SetLineEndings'] = createExportWrapper(
      'EPDFAnnot_SetLineEndings',
      3,
    ));
    var _EPDFAnnot_GetLineEndings = (Module['_EPDFAnnot_GetLineEndings'] = createExportWrapper(
      'EPDFAnnot_GetLineEndings',
      3,
    ));
    var _EPDFAnnot_SetVertices = (Module['_EPDFAnnot_SetVertices'] = createExportWrapper(
      'EPDFAnnot_SetVertices',
      3,
    ));
    var _EPDFAnnot_SetLine = (Module['_EPDFAnnot_SetLine'] = createExportWrapper(
      'EPDFAnnot_SetLine',
      3,
    ));
    var _EPDFAnnot_SetDefaultAppearance = (Module['_EPDFAnnot_SetDefaultAppearance'] =
      createExportWrapper('EPDFAnnot_SetDefaultAppearance', 6));
    var _EPDFAnnot_GetDefaultAppearance = (Module['_EPDFAnnot_GetDefaultAppearance'] =
      createExportWrapper('EPDFAnnot_GetDefaultAppearance', 6));
    var _EPDFAnnot_SetTextAlignment = (Module['_EPDFAnnot_SetTextAlignment'] = createExportWrapper(
      'EPDFAnnot_SetTextAlignment',
      2,
    ));
    var _EPDFAnnot_GetTextAlignment = (Module['_EPDFAnnot_GetTextAlignment'] = createExportWrapper(
      'EPDFAnnot_GetTextAlignment',
      1,
    ));
    var _EPDFAnnot_SetVerticalAlignment = (Module['_EPDFAnnot_SetVerticalAlignment'] =
      createExportWrapper('EPDFAnnot_SetVerticalAlignment', 2));
    var _EPDFAnnot_GetVerticalAlignment = (Module['_EPDFAnnot_GetVerticalAlignment'] =
      createExportWrapper('EPDFAnnot_GetVerticalAlignment', 1));
    var _EPDFPage_GetAnnotByName = (Module['_EPDFPage_GetAnnotByName'] = createExportWrapper(
      'EPDFPage_GetAnnotByName',
      2,
    ));
    var _EPDFPage_RemoveAnnotByName = (Module['_EPDFPage_RemoveAnnotByName'] = createExportWrapper(
      'EPDFPage_RemoveAnnotByName',
      2,
    ));
    var _EPDFAnnot_SetLinkedAnnot = (Module['_EPDFAnnot_SetLinkedAnnot'] = createExportWrapper(
      'EPDFAnnot_SetLinkedAnnot',
      3,
    ));
    var _EPDFPage_GetAnnotCountRaw = (Module['_EPDFPage_GetAnnotCountRaw'] = createExportWrapper(
      'EPDFPage_GetAnnotCountRaw',
      2,
    ));
    var _EPDFPage_GetAnnotRaw = (Module['_EPDFPage_GetAnnotRaw'] = createExportWrapper(
      'EPDFPage_GetAnnotRaw',
      3,
    ));
    var _EPDFPage_RemoveAnnotRaw = (Module['_EPDFPage_RemoveAnnotRaw'] = createExportWrapper(
      'EPDFPage_RemoveAnnotRaw',
      3,
    ));
    var _EPDFAnnot_SetIcon = (Module['_EPDFAnnot_SetIcon'] = createExportWrapper(
      'EPDFAnnot_SetIcon',
      2,
    ));
    var _EPDFAnnot_GetIcon = (Module['_EPDFAnnot_GetIcon'] = createExportWrapper(
      'EPDFAnnot_GetIcon',
      1,
    ));
    var _EPDFAnnot_UpdateAppearanceToRect = (Module['_EPDFAnnot_UpdateAppearanceToRect'] =
      createExportWrapper('EPDFAnnot_UpdateAppearanceToRect', 2));
    var _EPDFPage_CreateAnnot = (Module['_EPDFPage_CreateAnnot'] = createExportWrapper(
      'EPDFPage_CreateAnnot',
      2,
    ));
    var _FPDFDoc_GetAttachmentCount = (Module['_FPDFDoc_GetAttachmentCount'] = createExportWrapper(
      'FPDFDoc_GetAttachmentCount',
      1,
    ));
    var _FPDFDoc_AddAttachment = (Module['_FPDFDoc_AddAttachment'] = createExportWrapper(
      'FPDFDoc_AddAttachment',
      2,
    ));
    var _FPDFDoc_GetAttachment = (Module['_FPDFDoc_GetAttachment'] = createExportWrapper(
      'FPDFDoc_GetAttachment',
      2,
    ));
    var _FPDFDoc_DeleteAttachment = (Module['_FPDFDoc_DeleteAttachment'] = createExportWrapper(
      'FPDFDoc_DeleteAttachment',
      2,
    ));
    var _FPDFAttachment_GetName = (Module['_FPDFAttachment_GetName'] = createExportWrapper(
      'FPDFAttachment_GetName',
      3,
    ));
    var _FPDFAttachment_HasKey = (Module['_FPDFAttachment_HasKey'] = createExportWrapper(
      'FPDFAttachment_HasKey',
      2,
    ));
    var _FPDFAttachment_GetValueType = (Module['_FPDFAttachment_GetValueType'] =
      createExportWrapper('FPDFAttachment_GetValueType', 2));
    var _FPDFAttachment_SetStringValue = (Module['_FPDFAttachment_SetStringValue'] =
      createExportWrapper('FPDFAttachment_SetStringValue', 3));
    var _FPDFAttachment_GetStringValue = (Module['_FPDFAttachment_GetStringValue'] =
      createExportWrapper('FPDFAttachment_GetStringValue', 4));
    var _FPDFAttachment_SetFile = (Module['_FPDFAttachment_SetFile'] = createExportWrapper(
      'FPDFAttachment_SetFile',
      4,
    ));
    var _FPDFAttachment_GetFile = (Module['_FPDFAttachment_GetFile'] = createExportWrapper(
      'FPDFAttachment_GetFile',
      4,
    ));
    var _FPDFAttachment_GetSubtype = (Module['_FPDFAttachment_GetSubtype'] = createExportWrapper(
      'FPDFAttachment_GetSubtype',
      3,
    ));
    var _EPDFAttachment_SetSubtype = (Module['_EPDFAttachment_SetSubtype'] = createExportWrapper(
      'EPDFAttachment_SetSubtype',
      2,
    ));
    var _EPDFAttachment_SetDescription = (Module['_EPDFAttachment_SetDescription'] =
      createExportWrapper('EPDFAttachment_SetDescription', 2));
    var _EPDFAttachment_GetDescription = (Module['_EPDFAttachment_GetDescription'] =
      createExportWrapper('EPDFAttachment_GetDescription', 3));
    var _EPDFAttachment_GetIntegerValue = (Module['_EPDFAttachment_GetIntegerValue'] =
      createExportWrapper('EPDFAttachment_GetIntegerValue', 3));
    var _FPDFCatalog_IsTagged = (Module['_FPDFCatalog_IsTagged'] = createExportWrapper(
      'FPDFCatalog_IsTagged',
      1,
    ));
    var _FPDFCatalog_SetLanguage = (Module['_FPDFCatalog_SetLanguage'] = createExportWrapper(
      'FPDFCatalog_SetLanguage',
      2,
    ));
    var _EPDFCatalog_GetLanguage = (Module['_EPDFCatalog_GetLanguage'] = createExportWrapper(
      'EPDFCatalog_GetLanguage',
      3,
    ));
    var _FPDFAvail_Create = (Module['_FPDFAvail_Create'] = createExportWrapper(
      'FPDFAvail_Create',
      2,
    ));
    var _FPDFAvail_Destroy = (Module['_FPDFAvail_Destroy'] = createExportWrapper(
      'FPDFAvail_Destroy',
      1,
    ));
    var _FPDFAvail_IsDocAvail = (Module['_FPDFAvail_IsDocAvail'] = createExportWrapper(
      'FPDFAvail_IsDocAvail',
      2,
    ));
    var _FPDFAvail_GetDocument = (Module['_FPDFAvail_GetDocument'] = createExportWrapper(
      'FPDFAvail_GetDocument',
      2,
    ));
    var _FPDFAvail_GetFirstPageNum = (Module['_FPDFAvail_GetFirstPageNum'] = createExportWrapper(
      'FPDFAvail_GetFirstPageNum',
      1,
    ));
    var _FPDFAvail_IsPageAvail = (Module['_FPDFAvail_IsPageAvail'] = createExportWrapper(
      'FPDFAvail_IsPageAvail',
      3,
    ));
    var _FPDFAvail_IsFormAvail = (Module['_FPDFAvail_IsFormAvail'] = createExportWrapper(
      'FPDFAvail_IsFormAvail',
      2,
    ));
    var _FPDFAvail_IsLinearized = (Module['_FPDFAvail_IsLinearized'] = createExportWrapper(
      'FPDFAvail_IsLinearized',
      1,
    ));
    var _FPDFBookmark_GetFirstChild = (Module['_FPDFBookmark_GetFirstChild'] = createExportWrapper(
      'FPDFBookmark_GetFirstChild',
      2,
    ));
    var _FPDFBookmark_GetNextSibling = (Module['_FPDFBookmark_GetNextSibling'] =
      createExportWrapper('FPDFBookmark_GetNextSibling', 2));
    var _FPDFBookmark_GetTitle = (Module['_FPDFBookmark_GetTitle'] = createExportWrapper(
      'FPDFBookmark_GetTitle',
      3,
    ));
    var _FPDFBookmark_GetCount = (Module['_FPDFBookmark_GetCount'] = createExportWrapper(
      'FPDFBookmark_GetCount',
      1,
    ));
    var _FPDFBookmark_Find = (Module['_FPDFBookmark_Find'] = createExportWrapper(
      'FPDFBookmark_Find',
      2,
    ));
    var _FPDFBookmark_GetDest = (Module['_FPDFBookmark_GetDest'] = createExportWrapper(
      'FPDFBookmark_GetDest',
      2,
    ));
    var _FPDFBookmark_GetAction = (Module['_FPDFBookmark_GetAction'] = createExportWrapper(
      'FPDFBookmark_GetAction',
      1,
    ));
    var _FPDFAction_GetType = (Module['_FPDFAction_GetType'] = createExportWrapper(
      'FPDFAction_GetType',
      1,
    ));
    var _FPDFAction_GetDest = (Module['_FPDFAction_GetDest'] = createExportWrapper(
      'FPDFAction_GetDest',
      2,
    ));
    var _FPDFAction_GetFilePath = (Module['_FPDFAction_GetFilePath'] = createExportWrapper(
      'FPDFAction_GetFilePath',
      3,
    ));
    var _FPDFAction_GetURIPath = (Module['_FPDFAction_GetURIPath'] = createExportWrapper(
      'FPDFAction_GetURIPath',
      4,
    ));
    var _FPDFDest_GetDestPageIndex = (Module['_FPDFDest_GetDestPageIndex'] = createExportWrapper(
      'FPDFDest_GetDestPageIndex',
      2,
    ));
    var _FPDFDest_GetView = (Module['_FPDFDest_GetView'] = createExportWrapper(
      'FPDFDest_GetView',
      3,
    ));
    var _FPDFDest_GetLocationInPage = (Module['_FPDFDest_GetLocationInPage'] = createExportWrapper(
      'FPDFDest_GetLocationInPage',
      7,
    ));
    var _FPDFLink_GetLinkAtPoint = (Module['_FPDFLink_GetLinkAtPoint'] = createExportWrapper(
      'FPDFLink_GetLinkAtPoint',
      3,
    ));
    var _FPDFLink_GetLinkZOrderAtPoint = (Module['_FPDFLink_GetLinkZOrderAtPoint'] =
      createExportWrapper('FPDFLink_GetLinkZOrderAtPoint', 3));
    var _FPDFLink_GetDest = (Module['_FPDFLink_GetDest'] = createExportWrapper(
      'FPDFLink_GetDest',
      2,
    ));
    var _FPDFLink_GetAction = (Module['_FPDFLink_GetAction'] = createExportWrapper(
      'FPDFLink_GetAction',
      1,
    ));
    var _FPDFLink_Enumerate = (Module['_FPDFLink_Enumerate'] = createExportWrapper(
      'FPDFLink_Enumerate',
      3,
    ));
    var _FPDFLink_GetAnnot = (Module['_FPDFLink_GetAnnot'] = createExportWrapper(
      'FPDFLink_GetAnnot',
      2,
    ));
    var _FPDFLink_GetAnnotRect = (Module['_FPDFLink_GetAnnotRect'] = createExportWrapper(
      'FPDFLink_GetAnnotRect',
      2,
    ));
    var _FPDFLink_CountQuadPoints = (Module['_FPDFLink_CountQuadPoints'] = createExportWrapper(
      'FPDFLink_CountQuadPoints',
      1,
    ));
    var _FPDFLink_GetQuadPoints = (Module['_FPDFLink_GetQuadPoints'] = createExportWrapper(
      'FPDFLink_GetQuadPoints',
      3,
    ));
    var _FPDF_GetPageAAction = (Module['_FPDF_GetPageAAction'] = createExportWrapper(
      'FPDF_GetPageAAction',
      2,
    ));
    var _FPDF_GetFileIdentifier = (Module['_FPDF_GetFileIdentifier'] = createExportWrapper(
      'FPDF_GetFileIdentifier',
      4,
    ));
    var _FPDF_GetMetaText = (Module['_FPDF_GetMetaText'] = createExportWrapper(
      'FPDF_GetMetaText',
      4,
    ));
    var _FPDF_GetPageLabel = (Module['_FPDF_GetPageLabel'] = createExportWrapper(
      'FPDF_GetPageLabel',
      4,
    ));
    var _EPDF_SetMetaText = (Module['_EPDF_SetMetaText'] = createExportWrapper(
      'EPDF_SetMetaText',
      3,
    ));
    var _EPDF_HasMetaText = (Module['_EPDF_HasMetaText'] = createExportWrapper(
      'EPDF_HasMetaText',
      2,
    ));
    var _EPDF_GetMetaTrapped = (Module['_EPDF_GetMetaTrapped'] = createExportWrapper(
      'EPDF_GetMetaTrapped',
      1,
    ));
    var _EPDF_SetMetaTrapped = (Module['_EPDF_SetMetaTrapped'] = createExportWrapper(
      'EPDF_SetMetaTrapped',
      2,
    ));
    var _EPDF_GetMetaKeyCount = (Module['_EPDF_GetMetaKeyCount'] = createExportWrapper(
      'EPDF_GetMetaKeyCount',
      2,
    ));
    var _EPDF_GetMetaKeyName = (Module['_EPDF_GetMetaKeyName'] = createExportWrapper(
      'EPDF_GetMetaKeyName',
      5,
    ));
    var _FPDFPageObj_NewImageObj = (Module['_FPDFPageObj_NewImageObj'] = createExportWrapper(
      'FPDFPageObj_NewImageObj',
      1,
    ));
    var _FPDFImageObj_LoadJpegFile = (Module['_FPDFImageObj_LoadJpegFile'] = createExportWrapper(
      'FPDFImageObj_LoadJpegFile',
      4,
    ));
    var _FPDFImageObj_LoadJpegFileInline = (Module['_FPDFImageObj_LoadJpegFileInline'] =
      createExportWrapper('FPDFImageObj_LoadJpegFileInline', 4));
    var _FPDFImageObj_SetMatrix = (Module['_FPDFImageObj_SetMatrix'] = createExportWrapper(
      'FPDFImageObj_SetMatrix',
      7,
    ));
    var _FPDFImageObj_SetBitmap = (Module['_FPDFImageObj_SetBitmap'] = createExportWrapper(
      'FPDFImageObj_SetBitmap',
      4,
    ));
    var _FPDFImageObj_GetBitmap = (Module['_FPDFImageObj_GetBitmap'] = createExportWrapper(
      'FPDFImageObj_GetBitmap',
      1,
    ));
    var _FPDFImageObj_GetRenderedBitmap = (Module['_FPDFImageObj_GetRenderedBitmap'] =
      createExportWrapper('FPDFImageObj_GetRenderedBitmap', 3));
    var _FPDFImageObj_GetImageDataDecoded = (Module['_FPDFImageObj_GetImageDataDecoded'] =
      createExportWrapper('FPDFImageObj_GetImageDataDecoded', 3));
    var _FPDFImageObj_GetImageDataRaw = (Module['_FPDFImageObj_GetImageDataRaw'] =
      createExportWrapper('FPDFImageObj_GetImageDataRaw', 3));
    var _FPDFImageObj_GetImageFilterCount = (Module['_FPDFImageObj_GetImageFilterCount'] =
      createExportWrapper('FPDFImageObj_GetImageFilterCount', 1));
    var _FPDFImageObj_GetImageFilter = (Module['_FPDFImageObj_GetImageFilter'] =
      createExportWrapper('FPDFImageObj_GetImageFilter', 4));
    var _FPDFImageObj_GetImageMetadata = (Module['_FPDFImageObj_GetImageMetadata'] =
      createExportWrapper('FPDFImageObj_GetImageMetadata', 3));
    var _FPDFImageObj_GetImagePixelSize = (Module['_FPDFImageObj_GetImagePixelSize'] =
      createExportWrapper('FPDFImageObj_GetImagePixelSize', 3));
    var _FPDFImageObj_GetIccProfileDataDecoded = (Module['_FPDFImageObj_GetIccProfileDataDecoded'] =
      createExportWrapper('FPDFImageObj_GetIccProfileDataDecoded', 5));
    var _FPDF_CreateNewDocument = (Module['_FPDF_CreateNewDocument'] = createExportWrapper(
      'FPDF_CreateNewDocument',
      0,
    ));
    var _FPDFPage_Delete = (Module['_FPDFPage_Delete'] = createExportWrapper('FPDFPage_Delete', 2));
    var _FPDF_MovePages = (Module['_FPDF_MovePages'] = createExportWrapper('FPDF_MovePages', 4));
    var _FPDFPage_New = (Module['_FPDFPage_New'] = createExportWrapper('FPDFPage_New', 4));
    var _FPDFPage_GetRotation = (Module['_FPDFPage_GetRotation'] = createExportWrapper(
      'FPDFPage_GetRotation',
      1,
    ));
    var _FPDFPage_InsertObject = (Module['_FPDFPage_InsertObject'] = createExportWrapper(
      'FPDFPage_InsertObject',
      2,
    ));
    var _FPDFPage_InsertObjectAtIndex = (Module['_FPDFPage_InsertObjectAtIndex'] =
      createExportWrapper('FPDFPage_InsertObjectAtIndex', 3));
    var _FPDFPage_RemoveObject = (Module['_FPDFPage_RemoveObject'] = createExportWrapper(
      'FPDFPage_RemoveObject',
      2,
    ));
    var _FPDFPage_CountObjects = (Module['_FPDFPage_CountObjects'] = createExportWrapper(
      'FPDFPage_CountObjects',
      1,
    ));
    var _FPDFPage_GetObject = (Module['_FPDFPage_GetObject'] = createExportWrapper(
      'FPDFPage_GetObject',
      2,
    ));
    var _FPDFPage_HasTransparency = (Module['_FPDFPage_HasTransparency'] = createExportWrapper(
      'FPDFPage_HasTransparency',
      1,
    ));
    var _FPDFPageObj_Destroy = (Module['_FPDFPageObj_Destroy'] = createExportWrapper(
      'FPDFPageObj_Destroy',
      1,
    ));
    var _FPDFPageObj_GetMarkedContentID = (Module['_FPDFPageObj_GetMarkedContentID'] =
      createExportWrapper('FPDFPageObj_GetMarkedContentID', 1));
    var _FPDFPageObj_CountMarks = (Module['_FPDFPageObj_CountMarks'] = createExportWrapper(
      'FPDFPageObj_CountMarks',
      1,
    ));
    var _FPDFPageObj_GetMark = (Module['_FPDFPageObj_GetMark'] = createExportWrapper(
      'FPDFPageObj_GetMark',
      2,
    ));
    var _FPDFPageObj_AddMark = (Module['_FPDFPageObj_AddMark'] = createExportWrapper(
      'FPDFPageObj_AddMark',
      2,
    ));
    var _FPDFPageObj_RemoveMark = (Module['_FPDFPageObj_RemoveMark'] = createExportWrapper(
      'FPDFPageObj_RemoveMark',
      2,
    ));
    var _FPDFPageObjMark_GetName = (Module['_FPDFPageObjMark_GetName'] = createExportWrapper(
      'FPDFPageObjMark_GetName',
      4,
    ));
    var _FPDFPageObjMark_CountParams = (Module['_FPDFPageObjMark_CountParams'] =
      createExportWrapper('FPDFPageObjMark_CountParams', 1));
    var _FPDFPageObjMark_GetParamKey = (Module['_FPDFPageObjMark_GetParamKey'] =
      createExportWrapper('FPDFPageObjMark_GetParamKey', 5));
    var _FPDFPageObjMark_GetParamValueType = (Module['_FPDFPageObjMark_GetParamValueType'] =
      createExportWrapper('FPDFPageObjMark_GetParamValueType', 2));
    var _FPDFPageObjMark_GetParamIntValue = (Module['_FPDFPageObjMark_GetParamIntValue'] =
      createExportWrapper('FPDFPageObjMark_GetParamIntValue', 3));
    var _FPDFPageObjMark_GetParamStringValue = (Module['_FPDFPageObjMark_GetParamStringValue'] =
      createExportWrapper('FPDFPageObjMark_GetParamStringValue', 5));
    var _FPDFPageObjMark_GetParamBlobValue = (Module['_FPDFPageObjMark_GetParamBlobValue'] =
      createExportWrapper('FPDFPageObjMark_GetParamBlobValue', 5));
    var _FPDFPageObj_HasTransparency = (Module['_FPDFPageObj_HasTransparency'] =
      createExportWrapper('FPDFPageObj_HasTransparency', 1));
    var _FPDFPageObjMark_SetIntParam = (Module['_FPDFPageObjMark_SetIntParam'] =
      createExportWrapper('FPDFPageObjMark_SetIntParam', 5));
    var _FPDFPageObjMark_SetStringParam = (Module['_FPDFPageObjMark_SetStringParam'] =
      createExportWrapper('FPDFPageObjMark_SetStringParam', 5));
    var _FPDFPageObjMark_SetBlobParam = (Module['_FPDFPageObjMark_SetBlobParam'] =
      createExportWrapper('FPDFPageObjMark_SetBlobParam', 6));
    var _FPDFPageObjMark_RemoveParam = (Module['_FPDFPageObjMark_RemoveParam'] =
      createExportWrapper('FPDFPageObjMark_RemoveParam', 3));
    var _FPDFPageObj_GetType = (Module['_FPDFPageObj_GetType'] = createExportWrapper(
      'FPDFPageObj_GetType',
      1,
    ));
    var _FPDFPageObj_GetIsActive = (Module['_FPDFPageObj_GetIsActive'] = createExportWrapper(
      'FPDFPageObj_GetIsActive',
      2,
    ));
    var _FPDFPageObj_SetIsActive = (Module['_FPDFPageObj_SetIsActive'] = createExportWrapper(
      'FPDFPageObj_SetIsActive',
      2,
    ));
    var _FPDFPage_GenerateContent = (Module['_FPDFPage_GenerateContent'] = createExportWrapper(
      'FPDFPage_GenerateContent',
      1,
    ));
    var _FPDFPageObj_Transform = (Module['_FPDFPageObj_Transform'] = createExportWrapper(
      'FPDFPageObj_Transform',
      7,
    ));
    var _FPDFPageObj_TransformF = (Module['_FPDFPageObj_TransformF'] = createExportWrapper(
      'FPDFPageObj_TransformF',
      2,
    ));
    var _FPDFPageObj_GetMatrix = (Module['_FPDFPageObj_GetMatrix'] = createExportWrapper(
      'FPDFPageObj_GetMatrix',
      2,
    ));
    var _FPDFPageObj_SetMatrix = (Module['_FPDFPageObj_SetMatrix'] = createExportWrapper(
      'FPDFPageObj_SetMatrix',
      2,
    ));
    var _FPDFPageObj_SetBlendMode = (Module['_FPDFPageObj_SetBlendMode'] = createExportWrapper(
      'FPDFPageObj_SetBlendMode',
      2,
    ));
    var _FPDFPage_TransformAnnots = (Module['_FPDFPage_TransformAnnots'] = createExportWrapper(
      'FPDFPage_TransformAnnots',
      7,
    ));
    var _FPDFPage_SetRotation = (Module['_FPDFPage_SetRotation'] = createExportWrapper(
      'FPDFPage_SetRotation',
      2,
    ));
    var _FPDFPageObj_SetFillColor = (Module['_FPDFPageObj_SetFillColor'] = createExportWrapper(
      'FPDFPageObj_SetFillColor',
      5,
    ));
    var _FPDFPageObj_GetFillColor = (Module['_FPDFPageObj_GetFillColor'] = createExportWrapper(
      'FPDFPageObj_GetFillColor',
      5,
    ));
    var _FPDFPageObj_GetBounds = (Module['_FPDFPageObj_GetBounds'] = createExportWrapper(
      'FPDFPageObj_GetBounds',
      5,
    ));
    var _FPDFPageObj_GetRotatedBounds = (Module['_FPDFPageObj_GetRotatedBounds'] =
      createExportWrapper('FPDFPageObj_GetRotatedBounds', 2));
    var _FPDFPageObj_SetStrokeColor = (Module['_FPDFPageObj_SetStrokeColor'] = createExportWrapper(
      'FPDFPageObj_SetStrokeColor',
      5,
    ));
    var _FPDFPageObj_GetStrokeColor = (Module['_FPDFPageObj_GetStrokeColor'] = createExportWrapper(
      'FPDFPageObj_GetStrokeColor',
      5,
    ));
    var _FPDFPageObj_SetStrokeWidth = (Module['_FPDFPageObj_SetStrokeWidth'] = createExportWrapper(
      'FPDFPageObj_SetStrokeWidth',
      2,
    ));
    var _FPDFPageObj_GetStrokeWidth = (Module['_FPDFPageObj_GetStrokeWidth'] = createExportWrapper(
      'FPDFPageObj_GetStrokeWidth',
      2,
    ));
    var _FPDFPageObj_GetLineJoin = (Module['_FPDFPageObj_GetLineJoin'] = createExportWrapper(
      'FPDFPageObj_GetLineJoin',
      1,
    ));
    var _FPDFPageObj_SetLineJoin = (Module['_FPDFPageObj_SetLineJoin'] = createExportWrapper(
      'FPDFPageObj_SetLineJoin',
      2,
    ));
    var _FPDFPageObj_GetLineCap = (Module['_FPDFPageObj_GetLineCap'] = createExportWrapper(
      'FPDFPageObj_GetLineCap',
      1,
    ));
    var _FPDFPageObj_SetLineCap = (Module['_FPDFPageObj_SetLineCap'] = createExportWrapper(
      'FPDFPageObj_SetLineCap',
      2,
    ));
    var _FPDFPageObj_GetDashPhase = (Module['_FPDFPageObj_GetDashPhase'] = createExportWrapper(
      'FPDFPageObj_GetDashPhase',
      2,
    ));
    var _FPDFPageObj_SetDashPhase = (Module['_FPDFPageObj_SetDashPhase'] = createExportWrapper(
      'FPDFPageObj_SetDashPhase',
      2,
    ));
    var _FPDFPageObj_GetDashCount = (Module['_FPDFPageObj_GetDashCount'] = createExportWrapper(
      'FPDFPageObj_GetDashCount',
      1,
    ));
    var _FPDFPageObj_GetDashArray = (Module['_FPDFPageObj_GetDashArray'] = createExportWrapper(
      'FPDFPageObj_GetDashArray',
      3,
    ));
    var _FPDFPageObj_SetDashArray = (Module['_FPDFPageObj_SetDashArray'] = createExportWrapper(
      'FPDFPageObj_SetDashArray',
      4,
    ));
    var _FPDFFormObj_CountObjects = (Module['_FPDFFormObj_CountObjects'] = createExportWrapper(
      'FPDFFormObj_CountObjects',
      1,
    ));
    var _FPDFFormObj_GetObject = (Module['_FPDFFormObj_GetObject'] = createExportWrapper(
      'FPDFFormObj_GetObject',
      2,
    ));
    var _FPDFFormObj_RemoveObject = (Module['_FPDFFormObj_RemoveObject'] = createExportWrapper(
      'FPDFFormObj_RemoveObject',
      2,
    ));
    var _FPDFPageObj_CreateNewPath = (Module['_FPDFPageObj_CreateNewPath'] = createExportWrapper(
      'FPDFPageObj_CreateNewPath',
      2,
    ));
    var _FPDFPageObj_CreateNewRect = (Module['_FPDFPageObj_CreateNewRect'] = createExportWrapper(
      'FPDFPageObj_CreateNewRect',
      4,
    ));
    var _FPDFPath_CountSegments = (Module['_FPDFPath_CountSegments'] = createExportWrapper(
      'FPDFPath_CountSegments',
      1,
    ));
    var _FPDFPath_GetPathSegment = (Module['_FPDFPath_GetPathSegment'] = createExportWrapper(
      'FPDFPath_GetPathSegment',
      2,
    ));
    var _FPDFPath_MoveTo = (Module['_FPDFPath_MoveTo'] = createExportWrapper('FPDFPath_MoveTo', 3));
    var _FPDFPath_LineTo = (Module['_FPDFPath_LineTo'] = createExportWrapper('FPDFPath_LineTo', 3));
    var _FPDFPath_BezierTo = (Module['_FPDFPath_BezierTo'] = createExportWrapper(
      'FPDFPath_BezierTo',
      7,
    ));
    var _FPDFPath_Close = (Module['_FPDFPath_Close'] = createExportWrapper('FPDFPath_Close', 1));
    var _FPDFPath_SetDrawMode = (Module['_FPDFPath_SetDrawMode'] = createExportWrapper(
      'FPDFPath_SetDrawMode',
      3,
    ));
    var _FPDFPath_GetDrawMode = (Module['_FPDFPath_GetDrawMode'] = createExportWrapper(
      'FPDFPath_GetDrawMode',
      3,
    ));
    var _FPDFPathSegment_GetPoint = (Module['_FPDFPathSegment_GetPoint'] = createExportWrapper(
      'FPDFPathSegment_GetPoint',
      3,
    ));
    var _FPDFPathSegment_GetType = (Module['_FPDFPathSegment_GetType'] = createExportWrapper(
      'FPDFPathSegment_GetType',
      1,
    ));
    var _FPDFPathSegment_GetClose = (Module['_FPDFPathSegment_GetClose'] = createExportWrapper(
      'FPDFPathSegment_GetClose',
      1,
    ));
    var _FPDFPageObj_NewTextObj = (Module['_FPDFPageObj_NewTextObj'] = createExportWrapper(
      'FPDFPageObj_NewTextObj',
      3,
    ));
    var _FPDFText_SetText = (Module['_FPDFText_SetText'] = createExportWrapper(
      'FPDFText_SetText',
      2,
    ));
    var _FPDFText_SetCharcodes = (Module['_FPDFText_SetCharcodes'] = createExportWrapper(
      'FPDFText_SetCharcodes',
      3,
    ));
    var _FPDFText_LoadFont = (Module['_FPDFText_LoadFont'] = createExportWrapper(
      'FPDFText_LoadFont',
      5,
    ));
    var _FPDFText_LoadStandardFont = (Module['_FPDFText_LoadStandardFont'] = createExportWrapper(
      'FPDFText_LoadStandardFont',
      2,
    ));
    var _FPDFText_LoadCidType2Font = (Module['_FPDFText_LoadCidType2Font'] = createExportWrapper(
      'FPDFText_LoadCidType2Font',
      6,
    ));
    var _FPDFTextObj_GetFontSize = (Module['_FPDFTextObj_GetFontSize'] = createExportWrapper(
      'FPDFTextObj_GetFontSize',
      2,
    ));
    var _FPDFTextObj_GetText = (Module['_FPDFTextObj_GetText'] = createExportWrapper(
      'FPDFTextObj_GetText',
      4,
    ));
    var _FPDFTextObj_GetRenderedBitmap = (Module['_FPDFTextObj_GetRenderedBitmap'] =
      createExportWrapper('FPDFTextObj_GetRenderedBitmap', 4));
    var _FPDFFont_Close = (Module['_FPDFFont_Close'] = createExportWrapper('FPDFFont_Close', 1));
    var _FPDFPageObj_CreateTextObj = (Module['_FPDFPageObj_CreateTextObj'] = createExportWrapper(
      'FPDFPageObj_CreateTextObj',
      3,
    ));
    var _FPDFTextObj_GetTextRenderMode = (Module['_FPDFTextObj_GetTextRenderMode'] =
      createExportWrapper('FPDFTextObj_GetTextRenderMode', 1));
    var _FPDFTextObj_SetTextRenderMode = (Module['_FPDFTextObj_SetTextRenderMode'] =
      createExportWrapper('FPDFTextObj_SetTextRenderMode', 2));
    var _FPDFTextObj_GetFont = (Module['_FPDFTextObj_GetFont'] = createExportWrapper(
      'FPDFTextObj_GetFont',
      1,
    ));
    var _FPDFFont_GetBaseFontName = (Module['_FPDFFont_GetBaseFontName'] = createExportWrapper(
      'FPDFFont_GetBaseFontName',
      3,
    ));
    var _FPDFFont_GetFamilyName = (Module['_FPDFFont_GetFamilyName'] = createExportWrapper(
      'FPDFFont_GetFamilyName',
      3,
    ));
    var _FPDFFont_GetFontData = (Module['_FPDFFont_GetFontData'] = createExportWrapper(
      'FPDFFont_GetFontData',
      4,
    ));
    var _FPDFFont_GetIsEmbedded = (Module['_FPDFFont_GetIsEmbedded'] = createExportWrapper(
      'FPDFFont_GetIsEmbedded',
      1,
    ));
    var _FPDFFont_GetFlags = (Module['_FPDFFont_GetFlags'] = createExportWrapper(
      'FPDFFont_GetFlags',
      1,
    ));
    var _FPDFFont_GetWeight = (Module['_FPDFFont_GetWeight'] = createExportWrapper(
      'FPDFFont_GetWeight',
      1,
    ));
    var _FPDFFont_GetItalicAngle = (Module['_FPDFFont_GetItalicAngle'] = createExportWrapper(
      'FPDFFont_GetItalicAngle',
      2,
    ));
    var _FPDFFont_GetAscent = (Module['_FPDFFont_GetAscent'] = createExportWrapper(
      'FPDFFont_GetAscent',
      3,
    ));
    var _FPDFFont_GetDescent = (Module['_FPDFFont_GetDescent'] = createExportWrapper(
      'FPDFFont_GetDescent',
      3,
    ));
    var _FPDFFont_GetGlyphWidth = (Module['_FPDFFont_GetGlyphWidth'] = createExportWrapper(
      'FPDFFont_GetGlyphWidth',
      4,
    ));
    var _FPDFFont_GetGlyphPath = (Module['_FPDFFont_GetGlyphPath'] = createExportWrapper(
      'FPDFFont_GetGlyphPath',
      3,
    ));
    var _FPDFGlyphPath_CountGlyphSegments = (Module['_FPDFGlyphPath_CountGlyphSegments'] =
      createExportWrapper('FPDFGlyphPath_CountGlyphSegments', 1));
    var _FPDFGlyphPath_GetGlyphPathSegment = (Module['_FPDFGlyphPath_GetGlyphPathSegment'] =
      createExportWrapper('FPDFGlyphPath_GetGlyphPathSegment', 2));
    var _EPDFText_RedactInRect = (Module['_EPDFText_RedactInRect'] = createExportWrapper(
      'EPDFText_RedactInRect',
      4,
    ));
    var _EPDFText_RedactInQuads = (Module['_EPDFText_RedactInQuads'] = createExportWrapper(
      'EPDFText_RedactInQuads',
      5,
    ));
    var _FPDFDoc_GetPageMode = (Module['_FPDFDoc_GetPageMode'] = createExportWrapper(
      'FPDFDoc_GetPageMode',
      1,
    ));
    var _FPDFPage_Flatten = (Module['_FPDFPage_Flatten'] = createExportWrapper(
      'FPDFPage_Flatten',
      2,
    ));
    var _FPDFPage_HasFormFieldAtPoint = (Module['_FPDFPage_HasFormFieldAtPoint'] =
      createExportWrapper('FPDFPage_HasFormFieldAtPoint', 4));
    var _FPDFPage_FormFieldZOrderAtPoint = (Module['_FPDFPage_FormFieldZOrderAtPoint'] =
      createExportWrapper('FPDFPage_FormFieldZOrderAtPoint', 4));
    var _malloc = (Module['_malloc'] = createExportWrapper('malloc', 1));
    var _free = (Module['_free'] = createExportWrapper('free', 1));
    var _FORM_OnMouseMove = (Module['_FORM_OnMouseMove'] = createExportWrapper(
      'FORM_OnMouseMove',
      5,
    ));
    var _FORM_OnMouseWheel = (Module['_FORM_OnMouseWheel'] = createExportWrapper(
      'FORM_OnMouseWheel',
      6,
    ));
    var _FORM_OnFocus = (Module['_FORM_OnFocus'] = createExportWrapper('FORM_OnFocus', 5));
    var _FORM_OnLButtonDown = (Module['_FORM_OnLButtonDown'] = createExportWrapper(
      'FORM_OnLButtonDown',
      5,
    ));
    var _FORM_OnLButtonUp = (Module['_FORM_OnLButtonUp'] = createExportWrapper(
      'FORM_OnLButtonUp',
      5,
    ));
    var _FORM_OnLButtonDoubleClick = (Module['_FORM_OnLButtonDoubleClick'] = createExportWrapper(
      'FORM_OnLButtonDoubleClick',
      5,
    ));
    var _FORM_OnRButtonDown = (Module['_FORM_OnRButtonDown'] = createExportWrapper(
      'FORM_OnRButtonDown',
      5,
    ));
    var _FORM_OnRButtonUp = (Module['_FORM_OnRButtonUp'] = createExportWrapper(
      'FORM_OnRButtonUp',
      5,
    ));
    var _FORM_OnKeyDown = (Module['_FORM_OnKeyDown'] = createExportWrapper('FORM_OnKeyDown', 4));
    var _FORM_OnKeyUp = (Module['_FORM_OnKeyUp'] = createExportWrapper('FORM_OnKeyUp', 4));
    var _FORM_OnChar = (Module['_FORM_OnChar'] = createExportWrapper('FORM_OnChar', 4));
    var _FORM_GetFocusedText = (Module['_FORM_GetFocusedText'] = createExportWrapper(
      'FORM_GetFocusedText',
      4,
    ));
    var _FORM_GetSelectedText = (Module['_FORM_GetSelectedText'] = createExportWrapper(
      'FORM_GetSelectedText',
      4,
    ));
    var _FORM_ReplaceAndKeepSelection = (Module['_FORM_ReplaceAndKeepSelection'] =
      createExportWrapper('FORM_ReplaceAndKeepSelection', 3));
    var _FORM_ReplaceSelection = (Module['_FORM_ReplaceSelection'] = createExportWrapper(
      'FORM_ReplaceSelection',
      3,
    ));
    var _FORM_SelectAllText = (Module['_FORM_SelectAllText'] = createExportWrapper(
      'FORM_SelectAllText',
      2,
    ));
    var _FORM_CanUndo = (Module['_FORM_CanUndo'] = createExportWrapper('FORM_CanUndo', 2));
    var _FORM_CanRedo = (Module['_FORM_CanRedo'] = createExportWrapper('FORM_CanRedo', 2));
    var _FORM_Undo = (Module['_FORM_Undo'] = createExportWrapper('FORM_Undo', 2));
    var _FORM_Redo = (Module['_FORM_Redo'] = createExportWrapper('FORM_Redo', 2));
    var _FORM_ForceToKillFocus = (Module['_FORM_ForceToKillFocus'] = createExportWrapper(
      'FORM_ForceToKillFocus',
      1,
    ));
    var _FORM_GetFocusedAnnot = (Module['_FORM_GetFocusedAnnot'] = createExportWrapper(
      'FORM_GetFocusedAnnot',
      3,
    ));
    var _FORM_SetFocusedAnnot = (Module['_FORM_SetFocusedAnnot'] = createExportWrapper(
      'FORM_SetFocusedAnnot',
      2,
    ));
    var _FPDF_FFLDraw = (Module['_FPDF_FFLDraw'] = createExportWrapper('FPDF_FFLDraw', 9));
    var _FPDF_SetFormFieldHighlightColor = (Module['_FPDF_SetFormFieldHighlightColor'] =
      createExportWrapper('FPDF_SetFormFieldHighlightColor', 3));
    var _FPDF_SetFormFieldHighlightAlpha = (Module['_FPDF_SetFormFieldHighlightAlpha'] =
      createExportWrapper('FPDF_SetFormFieldHighlightAlpha', 2));
    var _FPDF_RemoveFormFieldHighlight = (Module['_FPDF_RemoveFormFieldHighlight'] =
      createExportWrapper('FPDF_RemoveFormFieldHighlight', 1));
    var _FORM_OnAfterLoadPage = (Module['_FORM_OnAfterLoadPage'] = createExportWrapper(
      'FORM_OnAfterLoadPage',
      2,
    ));
    var _FORM_OnBeforeClosePage = (Module['_FORM_OnBeforeClosePage'] = createExportWrapper(
      'FORM_OnBeforeClosePage',
      2,
    ));
    var _FORM_DoDocumentJSAction = (Module['_FORM_DoDocumentJSAction'] = createExportWrapper(
      'FORM_DoDocumentJSAction',
      1,
    ));
    var _FORM_DoDocumentOpenAction = (Module['_FORM_DoDocumentOpenAction'] = createExportWrapper(
      'FORM_DoDocumentOpenAction',
      1,
    ));
    var _FORM_DoDocumentAAction = (Module['_FORM_DoDocumentAAction'] = createExportWrapper(
      'FORM_DoDocumentAAction',
      2,
    ));
    var _FORM_DoPageAAction = (Module['_FORM_DoPageAAction'] = createExportWrapper(
      'FORM_DoPageAAction',
      3,
    ));
    var _FORM_SetIndexSelected = (Module['_FORM_SetIndexSelected'] = createExportWrapper(
      'FORM_SetIndexSelected',
      4,
    ));
    var _FORM_IsIndexSelected = (Module['_FORM_IsIndexSelected'] = createExportWrapper(
      'FORM_IsIndexSelected',
      3,
    ));
    var _FPDFDoc_GetJavaScriptActionCount = (Module['_FPDFDoc_GetJavaScriptActionCount'] =
      createExportWrapper('FPDFDoc_GetJavaScriptActionCount', 1));
    var _FPDFDoc_GetJavaScriptAction = (Module['_FPDFDoc_GetJavaScriptAction'] =
      createExportWrapper('FPDFDoc_GetJavaScriptAction', 2));
    var _FPDFDoc_CloseJavaScriptAction = (Module['_FPDFDoc_CloseJavaScriptAction'] =
      createExportWrapper('FPDFDoc_CloseJavaScriptAction', 1));
    var _FPDFJavaScriptAction_GetName = (Module['_FPDFJavaScriptAction_GetName'] =
      createExportWrapper('FPDFJavaScriptAction_GetName', 3));
    var _FPDFJavaScriptAction_GetScript = (Module['_FPDFJavaScriptAction_GetScript'] =
      createExportWrapper('FPDFJavaScriptAction_GetScript', 3));
    var _FPDF_ImportPagesByIndex = (Module['_FPDF_ImportPagesByIndex'] = createExportWrapper(
      'FPDF_ImportPagesByIndex',
      5,
    ));
    var _FPDF_ImportPages = (Module['_FPDF_ImportPages'] = createExportWrapper(
      'FPDF_ImportPages',
      4,
    ));
    var _FPDF_ImportNPagesToOne = (Module['_FPDF_ImportNPagesToOne'] = createExportWrapper(
      'FPDF_ImportNPagesToOne',
      5,
    ));
    var _FPDF_NewXObjectFromPage = (Module['_FPDF_NewXObjectFromPage'] = createExportWrapper(
      'FPDF_NewXObjectFromPage',
      3,
    ));
    var _FPDF_CloseXObject = (Module['_FPDF_CloseXObject'] = createExportWrapper(
      'FPDF_CloseXObject',
      1,
    ));
    var _FPDF_NewFormObjectFromXObject = (Module['_FPDF_NewFormObjectFromXObject'] =
      createExportWrapper('FPDF_NewFormObjectFromXObject', 1));
    var _FPDF_CopyViewerPreferences = (Module['_FPDF_CopyViewerPreferences'] = createExportWrapper(
      'FPDF_CopyViewerPreferences',
      2,
    ));
    var _FPDF_RenderPageBitmapWithColorScheme_Start = (Module[
      '_FPDF_RenderPageBitmapWithColorScheme_Start'
    ] = createExportWrapper('FPDF_RenderPageBitmapWithColorScheme_Start', 10));
    var _FPDF_RenderPageBitmap_Start = (Module['_FPDF_RenderPageBitmap_Start'] =
      createExportWrapper('FPDF_RenderPageBitmap_Start', 9));
    var _FPDF_RenderPage_Continue = (Module['_FPDF_RenderPage_Continue'] = createExportWrapper(
      'FPDF_RenderPage_Continue',
      2,
    ));
    var _FPDF_RenderPage_Close = (Module['_FPDF_RenderPage_Close'] = createExportWrapper(
      'FPDF_RenderPage_Close',
      1,
    ));
    var _FPDF_SaveWithVersion = (Module['_FPDF_SaveWithVersion'] = createExportWrapper(
      'FPDF_SaveWithVersion',
      4,
    ));
    var _FPDFText_GetCharIndexFromTextIndex = (Module['_FPDFText_GetCharIndexFromTextIndex'] =
      createExportWrapper('FPDFText_GetCharIndexFromTextIndex', 2));
    var _FPDFText_GetTextIndexFromCharIndex = (Module['_FPDFText_GetTextIndexFromCharIndex'] =
      createExportWrapper('FPDFText_GetTextIndexFromCharIndex', 2));
    var _FPDF_GetSignatureCount = (Module['_FPDF_GetSignatureCount'] = createExportWrapper(
      'FPDF_GetSignatureCount',
      1,
    ));
    var _FPDF_GetSignatureObject = (Module['_FPDF_GetSignatureObject'] = createExportWrapper(
      'FPDF_GetSignatureObject',
      2,
    ));
    var _FPDFSignatureObj_GetContents = (Module['_FPDFSignatureObj_GetContents'] =
      createExportWrapper('FPDFSignatureObj_GetContents', 3));
    var _FPDFSignatureObj_GetByteRange = (Module['_FPDFSignatureObj_GetByteRange'] =
      createExportWrapper('FPDFSignatureObj_GetByteRange', 3));
    var _FPDFSignatureObj_GetSubFilter = (Module['_FPDFSignatureObj_GetSubFilter'] =
      createExportWrapper('FPDFSignatureObj_GetSubFilter', 3));
    var _FPDFSignatureObj_GetReason = (Module['_FPDFSignatureObj_GetReason'] = createExportWrapper(
      'FPDFSignatureObj_GetReason',
      3,
    ));
    var _FPDFSignatureObj_GetTime = (Module['_FPDFSignatureObj_GetTime'] = createExportWrapper(
      'FPDFSignatureObj_GetTime',
      3,
    ));
    var _FPDFSignatureObj_GetDocMDPPermission = (Module['_FPDFSignatureObj_GetDocMDPPermission'] =
      createExportWrapper('FPDFSignatureObj_GetDocMDPPermission', 1));
    var _FPDF_StructTree_GetForPage = (Module['_FPDF_StructTree_GetForPage'] = createExportWrapper(
      'FPDF_StructTree_GetForPage',
      1,
    ));
    var _FPDF_StructTree_Close = (Module['_FPDF_StructTree_Close'] = createExportWrapper(
      'FPDF_StructTree_Close',
      1,
    ));
    var _FPDF_StructTree_CountChildren = (Module['_FPDF_StructTree_CountChildren'] =
      createExportWrapper('FPDF_StructTree_CountChildren', 1));
    var _FPDF_StructTree_GetChildAtIndex = (Module['_FPDF_StructTree_GetChildAtIndex'] =
      createExportWrapper('FPDF_StructTree_GetChildAtIndex', 2));
    var _FPDF_StructElement_GetAltText = (Module['_FPDF_StructElement_GetAltText'] =
      createExportWrapper('FPDF_StructElement_GetAltText', 3));
    var _FPDF_StructElement_GetActualText = (Module['_FPDF_StructElement_GetActualText'] =
      createExportWrapper('FPDF_StructElement_GetActualText', 3));
    var _FPDF_StructElement_GetID = (Module['_FPDF_StructElement_GetID'] = createExportWrapper(
      'FPDF_StructElement_GetID',
      3,
    ));
    var _FPDF_StructElement_GetLang = (Module['_FPDF_StructElement_GetLang'] = createExportWrapper(
      'FPDF_StructElement_GetLang',
      3,
    ));
    var _FPDF_StructElement_GetAttributeCount = (Module['_FPDF_StructElement_GetAttributeCount'] =
      createExportWrapper('FPDF_StructElement_GetAttributeCount', 1));
    var _FPDF_StructElement_GetAttributeAtIndex = (Module[
      '_FPDF_StructElement_GetAttributeAtIndex'
    ] = createExportWrapper('FPDF_StructElement_GetAttributeAtIndex', 2));
    var _FPDF_StructElement_GetStringAttribute = (Module['_FPDF_StructElement_GetStringAttribute'] =
      createExportWrapper('FPDF_StructElement_GetStringAttribute', 4));
    var _FPDF_StructElement_GetMarkedContentID = (Module['_FPDF_StructElement_GetMarkedContentID'] =
      createExportWrapper('FPDF_StructElement_GetMarkedContentID', 1));
    var _FPDF_StructElement_GetType = (Module['_FPDF_StructElement_GetType'] = createExportWrapper(
      'FPDF_StructElement_GetType',
      3,
    ));
    var _FPDF_StructElement_GetObjType = (Module['_FPDF_StructElement_GetObjType'] =
      createExportWrapper('FPDF_StructElement_GetObjType', 3));
    var _FPDF_StructElement_GetTitle = (Module['_FPDF_StructElement_GetTitle'] =
      createExportWrapper('FPDF_StructElement_GetTitle', 3));
    var _FPDF_StructElement_CountChildren = (Module['_FPDF_StructElement_CountChildren'] =
      createExportWrapper('FPDF_StructElement_CountChildren', 1));
    var _FPDF_StructElement_GetChildAtIndex = (Module['_FPDF_StructElement_GetChildAtIndex'] =
      createExportWrapper('FPDF_StructElement_GetChildAtIndex', 2));
    var _FPDF_StructElement_GetChildMarkedContentID = (Module[
      '_FPDF_StructElement_GetChildMarkedContentID'
    ] = createExportWrapper('FPDF_StructElement_GetChildMarkedContentID', 2));
    var _FPDF_StructElement_GetParent = (Module['_FPDF_StructElement_GetParent'] =
      createExportWrapper('FPDF_StructElement_GetParent', 1));
    var _FPDF_StructElement_Attr_GetCount = (Module['_FPDF_StructElement_Attr_GetCount'] =
      createExportWrapper('FPDF_StructElement_Attr_GetCount', 1));
    var _FPDF_StructElement_Attr_GetName = (Module['_FPDF_StructElement_Attr_GetName'] =
      createExportWrapper('FPDF_StructElement_Attr_GetName', 5));
    var _FPDF_StructElement_Attr_GetValue = (Module['_FPDF_StructElement_Attr_GetValue'] =
      createExportWrapper('FPDF_StructElement_Attr_GetValue', 2));
    var _FPDF_StructElement_Attr_GetType = (Module['_FPDF_StructElement_Attr_GetType'] =
      createExportWrapper('FPDF_StructElement_Attr_GetType', 1));
    var _FPDF_StructElement_Attr_GetBooleanValue = (Module[
      '_FPDF_StructElement_Attr_GetBooleanValue'
    ] = createExportWrapper('FPDF_StructElement_Attr_GetBooleanValue', 2));
    var _FPDF_StructElement_Attr_GetNumberValue = (Module[
      '_FPDF_StructElement_Attr_GetNumberValue'
    ] = createExportWrapper('FPDF_StructElement_Attr_GetNumberValue', 2));
    var _FPDF_StructElement_Attr_GetStringValue = (Module[
      '_FPDF_StructElement_Attr_GetStringValue'
    ] = createExportWrapper('FPDF_StructElement_Attr_GetStringValue', 4));
    var _FPDF_StructElement_Attr_GetBlobValue = (Module['_FPDF_StructElement_Attr_GetBlobValue'] =
      createExportWrapper('FPDF_StructElement_Attr_GetBlobValue', 4));
    var _FPDF_StructElement_Attr_CountChildren = (Module['_FPDF_StructElement_Attr_CountChildren'] =
      createExportWrapper('FPDF_StructElement_Attr_CountChildren', 1));
    var _FPDF_StructElement_Attr_GetChildAtIndex = (Module[
      '_FPDF_StructElement_Attr_GetChildAtIndex'
    ] = createExportWrapper('FPDF_StructElement_Attr_GetChildAtIndex', 2));
    var _FPDF_StructElement_GetMarkedContentIdCount = (Module[
      '_FPDF_StructElement_GetMarkedContentIdCount'
    ] = createExportWrapper('FPDF_StructElement_GetMarkedContentIdCount', 1));
    var _FPDF_StructElement_GetMarkedContentIdAtIndex = (Module[
      '_FPDF_StructElement_GetMarkedContentIdAtIndex'
    ] = createExportWrapper('FPDF_StructElement_GetMarkedContentIdAtIndex', 2));
    var _FPDF_AddInstalledFont = (Module['_FPDF_AddInstalledFont'] = createExportWrapper(
      'FPDF_AddInstalledFont',
      3,
    ));
    var _FPDF_SetSystemFontInfo = (Module['_FPDF_SetSystemFontInfo'] = createExportWrapper(
      'FPDF_SetSystemFontInfo',
      1,
    ));
    var _FPDF_GetDefaultTTFMap = (Module['_FPDF_GetDefaultTTFMap'] = createExportWrapper(
      'FPDF_GetDefaultTTFMap',
      0,
    ));
    var _FPDF_GetDefaultTTFMapCount = (Module['_FPDF_GetDefaultTTFMapCount'] = createExportWrapper(
      'FPDF_GetDefaultTTFMapCount',
      0,
    ));
    var _FPDF_GetDefaultTTFMapEntry = (Module['_FPDF_GetDefaultTTFMapEntry'] = createExportWrapper(
      'FPDF_GetDefaultTTFMapEntry',
      1,
    ));
    var _FPDF_GetDefaultSystemFontInfo = (Module['_FPDF_GetDefaultSystemFontInfo'] =
      createExportWrapper('FPDF_GetDefaultSystemFontInfo', 0));
    var _FPDF_FreeDefaultSystemFontInfo = (Module['_FPDF_FreeDefaultSystemFontInfo'] =
      createExportWrapper('FPDF_FreeDefaultSystemFontInfo', 1));
    var _FPDFText_LoadPage = (Module['_FPDFText_LoadPage'] = createExportWrapper(
      'FPDFText_LoadPage',
      1,
    ));
    var _FPDFText_ClosePage = (Module['_FPDFText_ClosePage'] = createExportWrapper(
      'FPDFText_ClosePage',
      1,
    ));
    var _FPDFText_CountChars = (Module['_FPDFText_CountChars'] = createExportWrapper(
      'FPDFText_CountChars',
      1,
    ));
    var _FPDFText_GetUnicode = (Module['_FPDFText_GetUnicode'] = createExportWrapper(
      'FPDFText_GetUnicode',
      2,
    ));
    var _FPDFText_GetTextObject = (Module['_FPDFText_GetTextObject'] = createExportWrapper(
      'FPDFText_GetTextObject',
      2,
    ));
    var _FPDFText_IsGenerated = (Module['_FPDFText_IsGenerated'] = createExportWrapper(
      'FPDFText_IsGenerated',
      2,
    ));
    var _FPDFText_IsHyphen = (Module['_FPDFText_IsHyphen'] = createExportWrapper(
      'FPDFText_IsHyphen',
      2,
    ));
    var _FPDFText_HasUnicodeMapError = (Module['_FPDFText_HasUnicodeMapError'] =
      createExportWrapper('FPDFText_HasUnicodeMapError', 2));
    var _FPDFText_GetFontSize = (Module['_FPDFText_GetFontSize'] = createExportWrapper(
      'FPDFText_GetFontSize',
      2,
    ));
    var _FPDFText_GetFontInfo = (Module['_FPDFText_GetFontInfo'] = createExportWrapper(
      'FPDFText_GetFontInfo',
      5,
    ));
    var _FPDFText_GetFontWeight = (Module['_FPDFText_GetFontWeight'] = createExportWrapper(
      'FPDFText_GetFontWeight',
      2,
    ));
    var _FPDFText_GetFillColor = (Module['_FPDFText_GetFillColor'] = createExportWrapper(
      'FPDFText_GetFillColor',
      6,
    ));
    var _FPDFText_GetStrokeColor = (Module['_FPDFText_GetStrokeColor'] = createExportWrapper(
      'FPDFText_GetStrokeColor',
      6,
    ));
    var _FPDFText_GetCharAngle = (Module['_FPDFText_GetCharAngle'] = createExportWrapper(
      'FPDFText_GetCharAngle',
      2,
    ));
    var _FPDFText_GetCharBox = (Module['_FPDFText_GetCharBox'] = createExportWrapper(
      'FPDFText_GetCharBox',
      6,
    ));
    var _FPDFText_GetLooseCharBox = (Module['_FPDFText_GetLooseCharBox'] = createExportWrapper(
      'FPDFText_GetLooseCharBox',
      3,
    ));
    var _FPDFText_GetMatrix = (Module['_FPDFText_GetMatrix'] = createExportWrapper(
      'FPDFText_GetMatrix',
      3,
    ));
    var _FPDFText_GetCharOrigin = (Module['_FPDFText_GetCharOrigin'] = createExportWrapper(
      'FPDFText_GetCharOrigin',
      4,
    ));
    var _FPDFText_GetCharIndexAtPos = (Module['_FPDFText_GetCharIndexAtPos'] = createExportWrapper(
      'FPDFText_GetCharIndexAtPos',
      5,
    ));
    var _FPDFText_GetText = (Module['_FPDFText_GetText'] = createExportWrapper(
      'FPDFText_GetText',
      4,
    ));
    var _FPDFText_CountRects = (Module['_FPDFText_CountRects'] = createExportWrapper(
      'FPDFText_CountRects',
      3,
    ));
    var _FPDFText_GetRect = (Module['_FPDFText_GetRect'] = createExportWrapper(
      'FPDFText_GetRect',
      6,
    ));
    var _FPDFText_GetBoundedText = (Module['_FPDFText_GetBoundedText'] = createExportWrapper(
      'FPDFText_GetBoundedText',
      7,
    ));
    var _FPDFText_FindStart = (Module['_FPDFText_FindStart'] = createExportWrapper(
      'FPDFText_FindStart',
      4,
    ));
    var _FPDFText_FindNext = (Module['_FPDFText_FindNext'] = createExportWrapper(
      'FPDFText_FindNext',
      1,
    ));
    var _FPDFText_FindPrev = (Module['_FPDFText_FindPrev'] = createExportWrapper(
      'FPDFText_FindPrev',
      1,
    ));
    var _FPDFText_GetSchResultIndex = (Module['_FPDFText_GetSchResultIndex'] = createExportWrapper(
      'FPDFText_GetSchResultIndex',
      1,
    ));
    var _FPDFText_GetSchCount = (Module['_FPDFText_GetSchCount'] = createExportWrapper(
      'FPDFText_GetSchCount',
      1,
    ));
    var _FPDFText_FindClose = (Module['_FPDFText_FindClose'] = createExportWrapper(
      'FPDFText_FindClose',
      1,
    ));
    var _FPDFLink_LoadWebLinks = (Module['_FPDFLink_LoadWebLinks'] = createExportWrapper(
      'FPDFLink_LoadWebLinks',
      1,
    ));
    var _FPDFLink_CountWebLinks = (Module['_FPDFLink_CountWebLinks'] = createExportWrapper(
      'FPDFLink_CountWebLinks',
      1,
    ));
    var _FPDFLink_GetURL = (Module['_FPDFLink_GetURL'] = createExportWrapper('FPDFLink_GetURL', 4));
    var _FPDFLink_CountRects = (Module['_FPDFLink_CountRects'] = createExportWrapper(
      'FPDFLink_CountRects',
      2,
    ));
    var _FPDFLink_GetRect = (Module['_FPDFLink_GetRect'] = createExportWrapper(
      'FPDFLink_GetRect',
      7,
    ));
    var _FPDFLink_GetTextRange = (Module['_FPDFLink_GetTextRange'] = createExportWrapper(
      'FPDFLink_GetTextRange',
      4,
    ));
    var _FPDFLink_CloseWebLinks = (Module['_FPDFLink_CloseWebLinks'] = createExportWrapper(
      'FPDFLink_CloseWebLinks',
      1,
    ));
    var _FPDFPage_GetDecodedThumbnailData = (Module['_FPDFPage_GetDecodedThumbnailData'] =
      createExportWrapper('FPDFPage_GetDecodedThumbnailData', 3));
    var _FPDFPage_GetRawThumbnailData = (Module['_FPDFPage_GetRawThumbnailData'] =
      createExportWrapper('FPDFPage_GetRawThumbnailData', 3));
    var _FPDFPage_GetThumbnailAsBitmap = (Module['_FPDFPage_GetThumbnailAsBitmap'] =
      createExportWrapper('FPDFPage_GetThumbnailAsBitmap', 1));
    var _FPDFPage_SetMediaBox = (Module['_FPDFPage_SetMediaBox'] = createExportWrapper(
      'FPDFPage_SetMediaBox',
      5,
    ));
    var _FPDFPage_SetCropBox = (Module['_FPDFPage_SetCropBox'] = createExportWrapper(
      'FPDFPage_SetCropBox',
      5,
    ));
    var _FPDFPage_SetBleedBox = (Module['_FPDFPage_SetBleedBox'] = createExportWrapper(
      'FPDFPage_SetBleedBox',
      5,
    ));
    var _FPDFPage_SetTrimBox = (Module['_FPDFPage_SetTrimBox'] = createExportWrapper(
      'FPDFPage_SetTrimBox',
      5,
    ));
    var _FPDFPage_SetArtBox = (Module['_FPDFPage_SetArtBox'] = createExportWrapper(
      'FPDFPage_SetArtBox',
      5,
    ));
    var _FPDFPage_GetMediaBox = (Module['_FPDFPage_GetMediaBox'] = createExportWrapper(
      'FPDFPage_GetMediaBox',
      5,
    ));
    var _FPDFPage_GetCropBox = (Module['_FPDFPage_GetCropBox'] = createExportWrapper(
      'FPDFPage_GetCropBox',
      5,
    ));
    var _FPDFPage_GetBleedBox = (Module['_FPDFPage_GetBleedBox'] = createExportWrapper(
      'FPDFPage_GetBleedBox',
      5,
    ));
    var _FPDFPage_GetTrimBox = (Module['_FPDFPage_GetTrimBox'] = createExportWrapper(
      'FPDFPage_GetTrimBox',
      5,
    ));
    var _FPDFPage_GetArtBox = (Module['_FPDFPage_GetArtBox'] = createExportWrapper(
      'FPDFPage_GetArtBox',
      5,
    ));
    var _FPDFPage_TransFormWithClip = (Module['_FPDFPage_TransFormWithClip'] = createExportWrapper(
      'FPDFPage_TransFormWithClip',
      3,
    ));
    var _FPDFPageObj_TransformClipPath = (Module['_FPDFPageObj_TransformClipPath'] =
      createExportWrapper('FPDFPageObj_TransformClipPath', 7));
    var _FPDFPageObj_GetClipPath = (Module['_FPDFPageObj_GetClipPath'] = createExportWrapper(
      'FPDFPageObj_GetClipPath',
      1,
    ));
    var _FPDFClipPath_CountPaths = (Module['_FPDFClipPath_CountPaths'] = createExportWrapper(
      'FPDFClipPath_CountPaths',
      1,
    ));
    var _FPDFClipPath_CountPathSegments = (Module['_FPDFClipPath_CountPathSegments'] =
      createExportWrapper('FPDFClipPath_CountPathSegments', 2));
    var _FPDFClipPath_GetPathSegment = (Module['_FPDFClipPath_GetPathSegment'] =
      createExportWrapper('FPDFClipPath_GetPathSegment', 3));
    var _FPDF_CreateClipPath = (Module['_FPDF_CreateClipPath'] = createExportWrapper(
      'FPDF_CreateClipPath',
      4,
    ));
    var _FPDF_DestroyClipPath = (Module['_FPDF_DestroyClipPath'] = createExportWrapper(
      'FPDF_DestroyClipPath',
      1,
    ));
    var _FPDFPage_InsertClipPath = (Module['_FPDFPage_InsertClipPath'] = createExportWrapper(
      'FPDFPage_InsertClipPath',
      2,
    ));
    var _FPDF_InitLibrary = (Module['_FPDF_InitLibrary'] = createExportWrapper(
      'FPDF_InitLibrary',
      0,
    ));
    var _FPDF_DestroyLibrary = (Module['_FPDF_DestroyLibrary'] = createExportWrapper(
      'FPDF_DestroyLibrary',
      0,
    ));
    var _FPDF_SetSandBoxPolicy = (Module['_FPDF_SetSandBoxPolicy'] = createExportWrapper(
      'FPDF_SetSandBoxPolicy',
      2,
    ));
    var _FPDF_LoadDocument = (Module['_FPDF_LoadDocument'] = createExportWrapper(
      'FPDF_LoadDocument',
      2,
    ));
    var _FPDF_GetFormType = (Module['_FPDF_GetFormType'] = createExportWrapper(
      'FPDF_GetFormType',
      1,
    ));
    var _FPDF_LoadXFA = (Module['_FPDF_LoadXFA'] = createExportWrapper('FPDF_LoadXFA', 1));
    var _FPDF_LoadMemDocument = (Module['_FPDF_LoadMemDocument'] = createExportWrapper(
      'FPDF_LoadMemDocument',
      3,
    ));
    var _FPDF_LoadMemDocument64 = (Module['_FPDF_LoadMemDocument64'] = createExportWrapper(
      'FPDF_LoadMemDocument64',
      3,
    ));
    var _FPDF_LoadCustomDocument = (Module['_FPDF_LoadCustomDocument'] = createExportWrapper(
      'FPDF_LoadCustomDocument',
      2,
    ));
    var _FPDF_GetFileVersion = (Module['_FPDF_GetFileVersion'] = createExportWrapper(
      'FPDF_GetFileVersion',
      2,
    ));
    var _FPDF_DocumentHasValidCrossReferenceTable = (Module[
      '_FPDF_DocumentHasValidCrossReferenceTable'
    ] = createExportWrapper('FPDF_DocumentHasValidCrossReferenceTable', 1));
    var _FPDF_GetDocPermissions = (Module['_FPDF_GetDocPermissions'] = createExportWrapper(
      'FPDF_GetDocPermissions',
      1,
    ));
    var _FPDF_GetDocUserPermissions = (Module['_FPDF_GetDocUserPermissions'] = createExportWrapper(
      'FPDF_GetDocUserPermissions',
      1,
    ));
    var _FPDF_GetSecurityHandlerRevision = (Module['_FPDF_GetSecurityHandlerRevision'] =
      createExportWrapper('FPDF_GetSecurityHandlerRevision', 1));
    var _FPDF_GetPageCount = (Module['_FPDF_GetPageCount'] = createExportWrapper(
      'FPDF_GetPageCount',
      1,
    ));
    var _FPDF_LoadPage = (Module['_FPDF_LoadPage'] = createExportWrapper('FPDF_LoadPage', 2));
    var _FPDF_GetPageWidthF = (Module['_FPDF_GetPageWidthF'] = createExportWrapper(
      'FPDF_GetPageWidthF',
      1,
    ));
    var _FPDF_GetPageWidth = (Module['_FPDF_GetPageWidth'] = createExportWrapper(
      'FPDF_GetPageWidth',
      1,
    ));
    var _FPDF_GetPageHeightF = (Module['_FPDF_GetPageHeightF'] = createExportWrapper(
      'FPDF_GetPageHeightF',
      1,
    ));
    var _FPDF_GetPageHeight = (Module['_FPDF_GetPageHeight'] = createExportWrapper(
      'FPDF_GetPageHeight',
      1,
    ));
    var _FPDF_GetPageBoundingBox = (Module['_FPDF_GetPageBoundingBox'] = createExportWrapper(
      'FPDF_GetPageBoundingBox',
      2,
    ));
    var _FPDF_RenderPageBitmap = (Module['_FPDF_RenderPageBitmap'] = createExportWrapper(
      'FPDF_RenderPageBitmap',
      8,
    ));
    var _FPDF_RenderPageBitmapWithMatrix = (Module['_FPDF_RenderPageBitmapWithMatrix'] =
      createExportWrapper('FPDF_RenderPageBitmapWithMatrix', 5));
    var _EPDF_RenderAnnotBitmap = (Module['_EPDF_RenderAnnotBitmap'] = createExportWrapper(
      'EPDF_RenderAnnotBitmap',
      6,
    ));
    var _FPDF_ClosePage = (Module['_FPDF_ClosePage'] = createExportWrapper('FPDF_ClosePage', 1));
    var _FPDF_CloseDocument = (Module['_FPDF_CloseDocument'] = createExportWrapper(
      'FPDF_CloseDocument',
      1,
    ));
    var _FPDF_GetLastError = (Module['_FPDF_GetLastError'] = createExportWrapper(
      'FPDF_GetLastError',
      0,
    ));
    var _FPDF_DeviceToPage = (Module['_FPDF_DeviceToPage'] = createExportWrapper(
      'FPDF_DeviceToPage',
      10,
    ));
    var _FPDF_PageToDevice = (Module['_FPDF_PageToDevice'] = createExportWrapper(
      'FPDF_PageToDevice',
      10,
    ));
    var _FPDFBitmap_Create = (Module['_FPDFBitmap_Create'] = createExportWrapper(
      'FPDFBitmap_Create',
      3,
    ));
    var _FPDFBitmap_CreateEx = (Module['_FPDFBitmap_CreateEx'] = createExportWrapper(
      'FPDFBitmap_CreateEx',
      5,
    ));
    var _FPDFBitmap_GetFormat = (Module['_FPDFBitmap_GetFormat'] = createExportWrapper(
      'FPDFBitmap_GetFormat',
      1,
    ));
    var _FPDFBitmap_FillRect = (Module['_FPDFBitmap_FillRect'] = createExportWrapper(
      'FPDFBitmap_FillRect',
      6,
    ));
    var _FPDFBitmap_GetBuffer = (Module['_FPDFBitmap_GetBuffer'] = createExportWrapper(
      'FPDFBitmap_GetBuffer',
      1,
    ));
    var _FPDFBitmap_GetWidth = (Module['_FPDFBitmap_GetWidth'] = createExportWrapper(
      'FPDFBitmap_GetWidth',
      1,
    ));
    var _FPDFBitmap_GetHeight = (Module['_FPDFBitmap_GetHeight'] = createExportWrapper(
      'FPDFBitmap_GetHeight',
      1,
    ));
    var _FPDFBitmap_GetStride = (Module['_FPDFBitmap_GetStride'] = createExportWrapper(
      'FPDFBitmap_GetStride',
      1,
    ));
    var _FPDFBitmap_Destroy = (Module['_FPDFBitmap_Destroy'] = createExportWrapper(
      'FPDFBitmap_Destroy',
      1,
    ));
    var _FPDF_GetPageSizeByIndexF = (Module['_FPDF_GetPageSizeByIndexF'] = createExportWrapper(
      'FPDF_GetPageSizeByIndexF',
      3,
    ));
    var _EPDF_GetPageRotationByIndex = (Module['_EPDF_GetPageRotationByIndex'] =
      createExportWrapper('EPDF_GetPageRotationByIndex', 2));
    var _FPDF_GetPageSizeByIndex = (Module['_FPDF_GetPageSizeByIndex'] = createExportWrapper(
      'FPDF_GetPageSizeByIndex',
      4,
    ));
    var _FPDF_VIEWERREF_GetPrintScaling = (Module['_FPDF_VIEWERREF_GetPrintScaling'] =
      createExportWrapper('FPDF_VIEWERREF_GetPrintScaling', 1));
    var _FPDF_VIEWERREF_GetNumCopies = (Module['_FPDF_VIEWERREF_GetNumCopies'] =
      createExportWrapper('FPDF_VIEWERREF_GetNumCopies', 1));
    var _FPDF_VIEWERREF_GetPrintPageRange = (Module['_FPDF_VIEWERREF_GetPrintPageRange'] =
      createExportWrapper('FPDF_VIEWERREF_GetPrintPageRange', 1));
    var _FPDF_VIEWERREF_GetPrintPageRangeCount = (Module['_FPDF_VIEWERREF_GetPrintPageRangeCount'] =
      createExportWrapper('FPDF_VIEWERREF_GetPrintPageRangeCount', 1));
    var _FPDF_VIEWERREF_GetPrintPageRangeElement = (Module[
      '_FPDF_VIEWERREF_GetPrintPageRangeElement'
    ] = createExportWrapper('FPDF_VIEWERREF_GetPrintPageRangeElement', 2));
    var _FPDF_VIEWERREF_GetDuplex = (Module['_FPDF_VIEWERREF_GetDuplex'] = createExportWrapper(
      'FPDF_VIEWERREF_GetDuplex',
      1,
    ));
    var _FPDF_VIEWERREF_GetName = (Module['_FPDF_VIEWERREF_GetName'] = createExportWrapper(
      'FPDF_VIEWERREF_GetName',
      4,
    ));
    var _FPDF_CountNamedDests = (Module['_FPDF_CountNamedDests'] = createExportWrapper(
      'FPDF_CountNamedDests',
      1,
    ));
    var _FPDF_GetNamedDestByName = (Module['_FPDF_GetNamedDestByName'] = createExportWrapper(
      'FPDF_GetNamedDestByName',
      2,
    ));
    var _FPDF_GetNamedDest = (Module['_FPDF_GetNamedDest'] = createExportWrapper(
      'FPDF_GetNamedDest',
      4,
    ));
    var _FPDF_GetXFAPacketCount = (Module['_FPDF_GetXFAPacketCount'] = createExportWrapper(
      'FPDF_GetXFAPacketCount',
      1,
    ));
    var _FPDF_GetXFAPacketName = (Module['_FPDF_GetXFAPacketName'] = createExportWrapper(
      'FPDF_GetXFAPacketName',
      4,
    ));
    var _FPDF_GetXFAPacketContent = (Module['_FPDF_GetXFAPacketContent'] = createExportWrapper(
      'FPDF_GetXFAPacketContent',
      5,
    ));
    var _FPDF_GetTrailerEnds = (Module['_FPDF_GetTrailerEnds'] = createExportWrapper(
      'FPDF_GetTrailerEnds',
      3,
    ));
    var _fflush = createExportWrapper('fflush', 1);
    var _emscripten_builtin_memalign = createExportWrapper('emscripten_builtin_memalign', 2);
    var _strerror = createExportWrapper('strerror', 1);
    var _setThrew = createExportWrapper('setThrew', 2);
    var __emscripten_tempret_set = createExportWrapper('_emscripten_tempret_set', 1);
    var _emscripten_stack_init = () =>
      (_emscripten_stack_init = wasmExports['emscripten_stack_init'])();
    var _emscripten_stack_get_free = () =>
      (_emscripten_stack_get_free = wasmExports['emscripten_stack_get_free'])();
    var _emscripten_stack_get_base = () =>
      (_emscripten_stack_get_base = wasmExports['emscripten_stack_get_base'])();
    var _emscripten_stack_get_end = () =>
      (_emscripten_stack_get_end = wasmExports['emscripten_stack_get_end'])();
    var __emscripten_stack_restore = (a0) =>
      (__emscripten_stack_restore = wasmExports['_emscripten_stack_restore'])(a0);
    var __emscripten_stack_alloc = (a0) =>
      (__emscripten_stack_alloc = wasmExports['_emscripten_stack_alloc'])(a0);
    var _emscripten_stack_get_current = () =>
      (_emscripten_stack_get_current = wasmExports['emscripten_stack_get_current'])();
    var dynCall_ji = (Module['dynCall_ji'] = createExportWrapper('dynCall_ji', 2));
    var dynCall_jij = (Module['dynCall_jij'] = createExportWrapper('dynCall_jij', 4));
    var dynCall_iiij = (Module['dynCall_iiij'] = createExportWrapper('dynCall_iiij', 5));
    var dynCall_iij = (Module['dynCall_iij'] = createExportWrapper('dynCall_iij', 4));
    var dynCall_j = (Module['dynCall_j'] = createExportWrapper('dynCall_j', 1));
    var dynCall_jji = (Module['dynCall_jji'] = createExportWrapper('dynCall_jji', 4));
    var dynCall_iji = (Module['dynCall_iji'] = createExportWrapper('dynCall_iji', 4));
    var dynCall_viijii = (Module['dynCall_viijii'] = createExportWrapper('dynCall_viijii', 7));
    var dynCall_iiji = (Module['dynCall_iiji'] = createExportWrapper('dynCall_iiji', 5));
    var dynCall_jiji = (Module['dynCall_jiji'] = createExportWrapper('dynCall_jiji', 5));
    var dynCall_iiiiij = (Module['dynCall_iiiiij'] = createExportWrapper('dynCall_iiiiij', 7));
    var dynCall_iiiiijj = (Module['dynCall_iiiiijj'] = createExportWrapper('dynCall_iiiiijj', 9));
    var dynCall_iiiiiijj = (Module['dynCall_iiiiiijj'] = createExportWrapper(
      'dynCall_iiiiiijj',
      10,
    ));
    var dynCall_viji = (Module['dynCall_viji'] = createExportWrapper('dynCall_viji', 5));

    function invoke_viii(index, a1, a2, a3) {
      var sp = stackSave();
      try {
        getWasmTableEntry(index)(a1, a2, a3);
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    function invoke_ii(index, a1) {
      var sp = stackSave();
      try {
        return getWasmTableEntry(index)(a1);
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    function invoke_iii(index, a1, a2) {
      var sp = stackSave();
      try {
        return getWasmTableEntry(index)(a1, a2);
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    function invoke_iiii(index, a1, a2, a3) {
      var sp = stackSave();
      try {
        return getWasmTableEntry(index)(a1, a2, a3);
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    function invoke_viiii(index, a1, a2, a3, a4) {
      var sp = stackSave();
      try {
        getWasmTableEntry(index)(a1, a2, a3, a4);
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    function invoke_iiiii(index, a1, a2, a3, a4) {
      var sp = stackSave();
      try {
        return getWasmTableEntry(index)(a1, a2, a3, a4);
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    function invoke_v(index) {
      var sp = stackSave();
      try {
        getWasmTableEntry(index)();
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    function invoke_vii(index, a1, a2) {
      var sp = stackSave();
      try {
        getWasmTableEntry(index)(a1, a2);
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    function invoke_viiiiiiiii(index, a1, a2, a3, a4, a5, a6, a7, a8, a9) {
      var sp = stackSave();
      try {
        getWasmTableEntry(index)(a1, a2, a3, a4, a5, a6, a7, a8, a9);
      } catch (e) {
        stackRestore(sp);
        if (e !== e + 0) throw e;
        _setThrew(1, 0);
      }
    }

    // include: postamble.js
    // === Auto-generated postamble setup entry stuff ===

    Module['wasmExports'] = wasmExports;
    Module['ccall'] = ccall;
    Module['cwrap'] = cwrap;
    Module['addFunction'] = addFunction;
    Module['removeFunction'] = removeFunction;
    Module['setValue'] = setValue;
    Module['getValue'] = getValue;
    Module['UTF8ToString'] = UTF8ToString;
    Module['stringToUTF8'] = stringToUTF8;
    Module['UTF16ToString'] = UTF16ToString;
    Module['stringToUTF16'] = stringToUTF16;
    var missingLibrarySymbols = [
      'writeI53ToI64',
      'writeI53ToI64Clamped',
      'writeI53ToI64Signaling',
      'writeI53ToU64Clamped',
      'writeI53ToU64Signaling',
      'readI53FromI64',
      'readI53FromU64',
      'convertI32PairToI53',
      'convertU32PairToI53',
      'getTempRet0',
      'setTempRet0',
      'exitJS',
      'inetPton4',
      'inetNtop4',
      'inetPton6',
      'inetNtop6',
      'readSockaddr',
      'writeSockaddr',
      'emscriptenLog',
      'readEmAsmArgs',
      'jstoi_q',
      'listenOnce',
      'autoResumeAudioContext',
      'dynCallLegacy',
      'getDynCaller',
      'dynCall',
      'handleException',
      'keepRuntimeAlive',
      'runtimeKeepalivePush',
      'runtimeKeepalivePop',
      'callUserCallback',
      'maybeExit',
      'asmjsMangle',
      'HandleAllocator',
      'getNativeTypeSize',
      'STACK_SIZE',
      'STACK_ALIGN',
      'POINTER_SIZE',
      'ASSERTIONS',
      'reallyNegative',
      'unSign',
      'strLen',
      'reSign',
      'formatString',
      'intArrayToString',
      'AsciiToString',
      'lengthBytesUTF16',
      'UTF32ToString',
      'stringToUTF32',
      'lengthBytesUTF32',
      'stringToNewUTF8',
      'registerKeyEventCallback',
      'maybeCStringToJsString',
      'findEventTarget',
      'getBoundingClientRect',
      'fillMouseEventData',
      'registerMouseEventCallback',
      'registerWheelEventCallback',
      'registerUiEventCallback',
      'registerFocusEventCallback',
      'fillDeviceOrientationEventData',
      'registerDeviceOrientationEventCallback',
      'fillDeviceMotionEventData',
      'registerDeviceMotionEventCallback',
      'screenOrientation',
      'fillOrientationChangeEventData',
      'registerOrientationChangeEventCallback',
      'fillFullscreenChangeEventData',
      'registerFullscreenChangeEventCallback',
      'JSEvents_requestFullscreen',
      'JSEvents_resizeCanvasForFullscreen',
      'registerRestoreOldStyle',
      'hideEverythingExceptGivenElement',
      'restoreHiddenElements',
      'setLetterbox',
      'softFullscreenResizeWebGLRenderTarget',
      'doRequestFullscreen',
      'fillPointerlockChangeEventData',
      'registerPointerlockChangeEventCallback',
      'registerPointerlockErrorEventCallback',
      'requestPointerLock',
      'fillVisibilityChangeEventData',
      'registerVisibilityChangeEventCallback',
      'registerTouchEventCallback',
      'fillGamepadEventData',
      'registerGamepadEventCallback',
      'registerBeforeUnloadEventCallback',
      'fillBatteryEventData',
      'battery',
      'registerBatteryEventCallback',
      'setCanvasElementSize',
      'getCanvasElementSize',
      'jsStackTrace',
      'getCallstack',
      'convertPCtoSourceLocation',
      'checkWasiClock',
      'wasiRightsToMuslOFlags',
      'wasiOFlagsToMuslOFlags',
      'createDyncallWrapper',
      'safeSetTimeout',
      'setImmediateWrapped',
      'clearImmediateWrapped',
      'polyfillSetImmediate',
      'registerPostMainLoop',
      'registerPreMainLoop',
      'getPromise',
      'makePromise',
      'idsToPromises',
      'makePromiseCallback',
      'ExceptionInfo',
      'findMatchingCatch',
      'Browser_asyncPrepareDataCounter',
      'safeRequestAnimationFrame',
      'arraySum',
      'addDays',
      'getSocketFromFD',
      'getSocketAddress',
      'FS_unlink',
      'FS_mkdirTree',
      '_setNetworkCallback',
      'heapObjectForWebGLType',
      'toTypedArrayIndex',
      'webgl_enable_ANGLE_instanced_arrays',
      'webgl_enable_OES_vertex_array_object',
      'webgl_enable_WEBGL_draw_buffers',
      'webgl_enable_WEBGL_multi_draw',
      'webgl_enable_EXT_polygon_offset_clamp',
      'webgl_enable_EXT_clip_control',
      'webgl_enable_WEBGL_polygon_mode',
      'emscriptenWebGLGet',
      'computeUnpackAlignedImageSize',
      'colorChannelsInGlTextureFormat',
      'emscriptenWebGLGetTexPixelData',
      'emscriptenWebGLGetUniform',
      'webglGetUniformLocation',
      'webglPrepareUniformLocationsBeforeFirstUse',
      'webglGetLeftBracePos',
      'emscriptenWebGLGetVertexAttrib',
      '__glGetActiveAttribOrUniform',
      'writeGLArray',
      'registerWebGlEventCallback',
      'runAndAbortIfError',
      'ALLOC_NORMAL',
      'ALLOC_STACK',
      'allocate',
      'writeStringToMemory',
      'writeAsciiToMemory',
      'setErrNo',
      'demangle',
      'stackTrace',
    ];
    missingLibrarySymbols.forEach(missingLibrarySymbol);

    var unexportedSymbols = [
      'run',
      'addOnPreRun',
      'addOnInit',
      'addOnPreMain',
      'addOnExit',
      'addOnPostRun',
      'addRunDependency',
      'removeRunDependency',
      'out',
      'err',
      'callMain',
      'abort',
      'wasmMemory',
      'writeStackCookie',
      'checkStackCookie',
      'convertI32PairToI53Checked',
      'stackSave',
      'stackRestore',
      'stackAlloc',
      'ptrToString',
      'zeroMemory',
      'getHeapMax',
      'growMemory',
      'ENV',
      'ERRNO_CODES',
      'strError',
      'DNS',
      'Protocols',
      'Sockets',
      'initRandomFill',
      'randomFill',
      'timers',
      'warnOnce',
      'readEmAsmArgsArray',
      'jstoi_s',
      'getExecutableName',
      'asyncLoad',
      'alignMemory',
      'mmapAlloc',
      'wasmTable',
      'noExitRuntime',
      'getCFunc',
      'uleb128Encode',
      'sigToWasmTypes',
      'generateFuncType',
      'convertJsFunctionToWasm',
      'freeTableIndexes',
      'functionsInTableMap',
      'getEmptyTableSlot',
      'updateTableMap',
      'getFunctionAddress',
      'PATH',
      'PATH_FS',
      'UTF8Decoder',
      'UTF8ArrayToString',
      'stringToUTF8Array',
      'lengthBytesUTF8',
      'intArrayFromString',
      'stringToAscii',
      'UTF16Decoder',
      'stringToUTF8OnStack',
      'writeArrayToMemory',
      'JSEvents',
      'specialHTMLTargets',
      'findCanvasEventTarget',
      'currentFullscreenStrategy',
      'restoreOldWindowedStyle',
      'UNWIND_CACHE',
      'ExitStatus',
      'getEnvStrings',
      'doReadv',
      'doWritev',
      'promiseMap',
      'uncaughtExceptionCount',
      'exceptionLast',
      'exceptionCaught',
      'Browser',
      'getPreloadedImageData__data',
      'wget',
      'MONTH_DAYS_REGULAR',
      'MONTH_DAYS_LEAP',
      'MONTH_DAYS_REGULAR_CUMULATIVE',
      'MONTH_DAYS_LEAP_CUMULATIVE',
      'isLeapYear',
      'ydayFromDate',
      'SYSCALLS',
      'preloadPlugins',
      'FS_createPreloadedFile',
      'FS_modeStringToFlags',
      'FS_getMode',
      'FS_stdin_getChar_buffer',
      'FS_stdin_getChar',
      'FS_createPath',
      'FS_createDevice',
      'FS_readFile',
      'FS',
      'FS_createDataFile',
      'FS_createLazyFile',
      'MEMFS',
      'TTY',
      'PIPEFS',
      'SOCKFS',
      'tempFixedLengthArray',
      'miniTempWebGLFloatBuffers',
      'miniTempWebGLIntBuffers',
      'GL',
      'AL',
      'GLUT',
      'EGL',
      'GLEW',
      'IDBStore',
      'SDL',
      'SDL_gfx',
      'allocateUTF8',
      'allocateUTF8OnStack',
      'print',
      'printErr',
    ];
    unexportedSymbols.forEach(unexportedRuntimeSymbol);

    var calledRun;
    var calledPrerun;

    dependenciesFulfilled = function runCaller() {
      // If run has never been called, and we should call run (INVOKE_RUN is true, and Module.noInitialRun is not false)
      if (!calledRun) run();
      if (!calledRun) dependenciesFulfilled = runCaller; // try this again later, after new deps are fulfilled
    };

    function stackCheckInit() {
      // This is normally called automatically during __wasm_call_ctors but need to
      // get these values before even running any of the ctors so we call it redundantly
      // here.
      _emscripten_stack_init();
      // TODO(sbc): Move writeStackCookie to native to to avoid this.
      writeStackCookie();
    }

    function run() {
      if (runDependencies > 0) {
        return;
      }

      stackCheckInit();

      if (!calledPrerun) {
        calledPrerun = 1;
        preRun();

        // a preRun added a dependency, run will be called later
        if (runDependencies > 0) {
          return;
        }
      }

      function doRun() {
        // run may have just been called through dependencies being fulfilled just in this very frame,
        // or while the async setStatus time below was happening
        if (calledRun) return;
        calledRun = 1;
        Module['calledRun'] = 1;

        if (ABORT) return;

        initRuntime();

        readyPromiseResolve(Module);
        Module['onRuntimeInitialized']?.();

        assert(
          !Module['_main'],
          'compiled without a main, but one is present. if you added it from JS, use Module["onRuntimeInitialized"]',
        );

        postRun();
      }

      if (Module['setStatus']) {
        Module['setStatus']('Running...');
        setTimeout(() => {
          setTimeout(() => Module['setStatus'](''), 1);
          doRun();
        }, 1);
      } else {
        doRun();
      }
      checkStackCookie();
    }

    function checkUnflushedContent() {
      // Compiler settings do not allow exiting the runtime, so flushing
      // the streams is not possible. but in ASSERTIONS mode we check
      // if there was something to flush, and if so tell the user they
      // should request that the runtime be exitable.
      // Normally we would not even include flush() at all, but in ASSERTIONS
      // builds we do so just for this check, and here we see if there is any
      // content to flush, that is, we check if there would have been
      // something a non-ASSERTIONS build would have not seen.
      // How we flush the streams depends on whether we are in SYSCALLS_REQUIRE_FILESYSTEM=0
      // mode (which has its own special function for this; otherwise, all
      // the code is inside libc)
      var oldOut = out;
      var oldErr = err;
      var has = false;
      out = err = (x) => {
        has = true;
      };
      try {
        // it doesn't matter if it fails
        _fflush(0);
        // also flush in the JS FS layer
        ['stdout', 'stderr'].forEach((name) => {
          var info = FS.analyzePath('/dev/' + name);
          if (!info) return;
          var stream = info.object;
          var rdev = stream.rdev;
          var tty = TTY.ttys[rdev];
          if (tty?.output?.length) {
            has = true;
          }
        });
      } catch (e) {}
      out = oldOut;
      err = oldErr;
      if (has) {
        warnOnce(
          'stdio streams had content in them that was not flushed. you should set EXIT_RUNTIME to 1 (see the Emscripten FAQ), or make sure to emit a newline when you printf etc.',
        );
      }
    }

    if (Module['preInit']) {
      if (typeof Module['preInit'] == 'function') Module['preInit'] = [Module['preInit']];
      while (Module['preInit'].length > 0) {
        Module['preInit'].pop()();
      }
    }

    run();

    // end include: postamble.js

    // include: postamble_modularize.js
    // In MODULARIZE mode we wrap the generated code in a factory function
    // and return either the Module itself, or a promise of the module.
    //
    // We assign to the `moduleRtn` global here and configure closure to see
    // this as and extern so it won't get minified.

    moduleRtn = readyPromise;

    // Assertion for attempting to access module properties on the incoming
    // moduleArg.  In the past we used this object as the prototype of the module
    // and assigned properties to it, but now we return a distinct object.  This
    // keeps the instance private until it is ready (i.e the promise has been
    // resolved).
    for (const prop of Object.keys(Module)) {
      if (!(prop in moduleArg)) {
        Object.defineProperty(moduleArg, prop, {
          configurable: true,
          get() {
            abort(
              `Access to module property ('${prop}') is no longer possible via the module constructor argument; Instead, use the result of the module constructor.`,
            );
          },
        });
      }
    }
    // end include: postamble_modularize.js

    return moduleRtn;
  };
})();
export default createPdfium;

==== FILE: C:\Users\charris\Downloads\embed-pdf-viewer-main\embed-pdf-viewer-main\packages\pdfium\src\vendor\pdfium.wasm ====
 asm   Æ­` ``` ``` ` `` `` `}`  ` `~` ``} ``~` `~`}}`	`}`}`~ `~~`~`	 `||`}}}} `
`|` ~`~`|`|`||`~~~~ `}`~ `}} `~`} ``~~`
 `~`}`}}}`~~`~ `}} `}}`} ``}}`|||| `|`||`}`|`~`~`~~`~`} `} `}} `}`||`	}}}}}}}} `|`|`}}}}}} `||`~~ `~~`|||||| ` |`~`~ `~`}`} `} `}`}`
 `}} `| `||`}`|}`|~~|`| `~`~`~~`|`~~~~`|`~~` `}`}`||`}}}}`}}`~ `~~`~ `} `}`}`	} `} `	}`}}}}}}`~~`~~`}}}`}` `
||`
}}}}}}}} `} `}}} `}}}}}} `}}}}}}} `||| `|`~` `}}`} `} `}`	}}} `}`}}} `}}`|||`~|`|||`|}`|~`~~|`~~`~~ `~~}`~ `||||`||||`||||||`}}}`}`	}}}}`}}}`}}} `}}}`}}`}}`}}}}}}`~~`~ `~`~~`~`~~`~ ô$envinvoke_viii env	invoke_ii env
invoke_iii envinvoke_iiii envinvoke_viiii 	envinvoke_iiiii envinvoke_v  env	_tzset_js env	_abort_js 
wasi_snapshot_preview1fd_close env_emscripten_memcpy_js envemscripten_date_now Qenv__syscall_openat env__syscall_fcntl64 env__syscall_ioctl wasi_snapshot_preview1fd_write wasi_snapshot_preview1fd_read env__syscall_fstat64 env__syscall_stat64 env__syscall_newfstatat env__syscall_lstat64 wasi_snapshot_preview1fd_sync wasi_snapshot_preview1environ_sizes_get wasi_snapshot_preview1environ_get env__syscall_getdents64 env__syscall_unlinkat env__syscall_rmdir envemscripten_resize_heap env_emscripten_throw_longjmp 
env
__assert_fail env
invoke_vii envinvoke_viiiiiiiii 0env__syscall_ftruncate64 wasi_snapshot_preview1fd_seek env
_localtime_js env
_gmtime_js ™LêK

     

      
   )&R 3  4        
 		  	S1  #
		                
	   	
			

S           51p     	1C    	 
	            ++6+++77  6D    $           

 q,
    
 T    
--   r8stEV	u						 v	  w U FE	!8 
 
  .9
  ! 6    - - 	               x

	    	  	 	  	             	-            
                                  D                                        	   
!     	    		            y
z /$   	                   D-WW  ::	{:: XGG	G    		          Y|  Y2         			      
      	

                           
 HH '
''''''       

   

  }
 
""	
  !~
    		     		           	
 [                                 			Z  
  
   
                 

          	      	     	 		    

    
         	

       	     II   	 + F€ F‚ƒI   \<

 =J  7=]„     % 	      )	"=…"

"J&"&K

 %%%%%%J .

$/ 5     V   .†
 .
  
     ‡ !         9              
                        	 	 											        																		             

   	 	 											            																		          													                

                     	 
 ˆ
	E‰	Š			‹Œ 6^                            >                   	            LL         Ž                    )


3))__^=""&`Qa   
3M  "&&)
&bb‘’`a73"“
	McdcMR$$	e”  N\(N•       ? 
		    5((ff((N(–(—1d˜™/
 <>K  


!

@







@

	9>

K





!











	9





,A
,A?gOh

,A
,A?gOh




		




		   .0 .0 B0i
B0i
 





























       
           

      
 
 
 
 
 
                             		    	
     @####@##<
 
   
 
     

					     	 	        <


š
›  œ    H  
PPj 

 2ž
Ÿ ¡¢
>2
j 2kkX2 		ll    £       
Pm  ¤mnn¥    5    C	  *        

 !         	   
1¦T§¨©ªB«¬!	Coop èœ€€A€€A A A Çv´memory __wasm_call_ctors $__indirect_function_table PDFiumExt_Init /FPDF_InitLibraryWithConfig ÏPDFiumExt_OpenFileWriter 0PDFiumExt_GetFileWriterSize 1PDFiumExt_GetFileWriterData 4PDFiumExt_CloseFileWriter 7PDFiumExt_SaveAsCopy 8FPDF_SaveAsCopy ®PDFiumExt_OpenFormFillInfo 9PDFiumExt_CloseFormFillInfo :!PDFiumExt_InitFormFillEnvironment ;FPDFDOC_InitFormFillEnvironment €!PDFiumExt_ExitFormFillEnvironment <FPDFDOC_ExitFormFillEnvironment EPDFNamedDest_SetDest ¯GEPDFNamedDest_Remove ±GEPDFDest_CreateView ²GEPDFDest_CreateXYZ µGEPDFDest_CreateRemoteView ·GEPDFDest_CreateRemoteXYZ ¸GEPDFAction_CreateGoTo ¹GEPDFAction_CreateGoToNamed ºGEPDFAction_CreateLaunch »G!EPDFAction_CreateRemoteGoToByName ¼GEPDFAction_CreateRemoteGoToDest ½GEPDFAction_CreateURI ¾GEPDFBookmark_Create ¿GEPDFBookmark_Delete ÃGEPDFBookmark_AppendChild ÅGEPDFBookmark_InsertAfter ÆGEPDFBookmark_Clear ÇGEPDFBookmark_SetTitle ÈGEPDFBookmark_SetDest ÉGEPDFBookmark_SetAction ÊGEPDFBookmark_ClearTarget ËGEPDF_PNG_EncodeRGBA êKFPDFAnnot_IsSupportedSubtype ·FFPDFPage_CreateAnnot ¸FFPDFPage_GetAnnotCount ¹FFPDFPage_GetAnnot ºFFPDFPage_GetAnnotIndex »FFPDFPage_CloseAnnot ¼FFPDFPage_RemoveAnnot ½FFPDFAnnot_GetSubtype ¾F"FPDFAnnot_IsObjectSupportedSubtype ¿FFPDFAnnot_UpdateObject ÀFFPDFAnnot_AddInkStroke ÂFFPDFAnnot_RemoveInkList ÃFFPDFAnnot_AppendObject ÄFFPDFAnnot_GetObjectCount ÅFFPDFAnnot_GetObject ÆFFPDFAnnot_RemoveObject ÇFFPDFAnnot_SetColor ÈFFPDFAnnot_GetColor ÉFFPDFAnnot_HasAttachmentPoints ÊFFPDFAnnot_SetAttachmentPoints ËF FPDFAnnot_AppendAttachmentPoints ÍFFPDFAnnot_CountAttachmentPoints ÎFFPDFAnnot_GetAttachmentPoints ÏFFPDFAnnot_SetRect ÐFFPDFAnnot_GetRect ÑFFPDFAnnot_GetVertices ÒFFPDFAnnot_GetInkListCount ÓFFPDFAnnot_GetInkListPath ÕFFPDFAnnot_GetLine ÖFFPDFAnnot_SetBorder ×FFPDFAnnot_GetBorder ØFFPDFAnnot_HasKey ÙFFPDFAnnot_GetValueType ÚFFPDFAnnot_SetStringValue ÛFFPDFAnnot_GetStringValue ÜFFPDFAnnot_GetNumberValue ÝFFPDFAnnot_SetAP ÞFFPDFAnnot_GetAP àFFPDFAnnot_GetLinkedAnnot áFFPDFAnnot_GetFlags âFFPDFAnnot_SetFlags ãFFPDFAnnot_GetFormFieldFlags äFFPDFAnnot_SetFormFieldFlags åFFPDFAnnot_GetFormFieldAtPoint æFFPDFAnnot_GetFormFieldName çFFPDFAnnot_GetFormFieldType èF+FPDFAnnot_GetFormAdditionalActionJavaScript éF#FPDFAnnot_GetFormFieldAlternateName êFFPDFAnnot_GetFormFieldValue ëFFPDFAnnot_GetOptionCount ìFFPDFAnnot_GetOptionLabel íFFPDFAnnot_IsOptionSelected îFFPDFAnnot_GetFontSize ïFFPDFAnnot_SetFontColor ñFFPDFAnnot_GetFontColor òFFPDFAnnot_IsChecked óFFPDFAnnot_SetFocusableSubtypes ôF#FPDFAnnot_GetFocusableSubtypesCount öFFPDFAnnot_GetFocusableSubtypes ÷FFPDFAnnot_GetLink øFFPDFAnnot_GetFormControlCount ùFFPDFAnnot_GetFormControlIndex úF!FPDFAnnot_GetFormFieldExportValue ûFFPDFAnnot_SetURI üFFPDFAnnot_GetFileAttachment ýFFPDFAnnot_AddFileAttachment þFEPDFAnnot_SetColor ÿFEPDFAnnot_GetColor €GEPDFAnnot_ClearColor GEPDFAnnot_SetOpacity ‚GEPDFAnnot_GetOpacity ƒGEPDFAnnot_GetBorderEffect „G!EPDFAnnot_GetRectangleDifferences …G#EPDFAnnot_GetBorderDashPatternCount †GEPDFAnnot_GetBorderDashPattern ‡GEPDFAnnot_SetBorderDashPattern ˆGEPDFAnnot_GetBorderStyle ŠGEPDFAnnot_SetBorderStyle ‹GEPDFAnnot_GenerateAppearance ŒG%EPDFAnnot_GenerateAppearanceWithBlend GEPDFAnnot_GetBlendMode ŽGEPDFAnnot_SetIntent GEPDFAnnot_GetIntent GEPDFAnnot_GetRichContent ‘GEPDFAnnot_SetLineEndings ’GEPDFAnnot_GetLineEndings ”GEPDFAnnot_SetVertices –GEPDFAnnot_SetLine —GEPDFAnnot_SetDefaultAppearance ˜GEPDFAnnot_GetDefaultAppearance ™GEPDFAnnot_SetTextAlignment šGEPDFAnnot_GetTextAlignment ›GEPDFAnnot_SetVerticalAlignment œGEPDFAnnot_GetVerticalAlignment GEPDFPage_GetAnnotByName žGEPDFPage_RemoveAnnotByName ŸGEPDFAnnot_SetLinkedAnnot  GEPDFPage_GetAnnotCountRaw £GEPDFPage_GetAnnotRaw ¤GEPDFPage_RemoveAnnotRaw ¨GEPDFAnnot_SetIcon ©GEPDFAnnot_GetIcon ªG EPDFAnnot_UpdateAppearanceToRect «GEPDFPage_CreateAnnot ®GFPDFDoc_GetAttachmentCount ®IFPDFDoc_AddAttachment ¯IFPDFDoc_GetAttachment °IFPDFDoc_DeleteAttachment ±IFPDFAttachment_GetName ²IFPDFAttachment_HasKey ³IFPDFAttachment_GetValueType ´IFPDFAttachment_SetStringValue µIFPDFAttachment_GetStringValue ·IFPDFAttachment_SetFile ¸IFPDFAttachment_GetFile ºIFPDFAttachment_GetSubtype »IEPDFAttachment_SetSubtype ¼IEPDFAttachment_SetDescription ½IEPDFAttachment_GetDescription ¾IEPDFAttachment_GetIntegerValue ¿IFPDFCatalog_IsTagged ˜KFPDFCatalog_SetLanguage ™KEPDFCatalog_GetLanguage šKFPDFAvail_Create ÌJFPDFAvail_Destroy ÎJFPDFAvail_IsDocAvail ÏJFPDFAvail_GetDocument ÑJFPDFAvail_GetFirstPageNum ÒJFPDFAvail_IsPageAvail ÓJFPDFAvail_IsFormAvail ÔJFPDFAvail_IsLinearized ÕJFPDFBookmark_GetFirstChild ­HFPDFBookmark_GetNextSibling ®HFPDFBookmark_GetTitle ¯HFPDFBookmark_GetCount °HFPDFBookmark_Find ±HFPDFBookmark_GetDest ³HFPDFBookmark_GetAction ´HFPDFAction_GetType µHFPDFAction_GetDest ¶HFPDFAction_GetFilePath ·HFPDFAction_GetURIPath ¸HFPDFDest_GetDestPageIndex ¹HFPDFDest_GetView ºHFPDFDest_GetLocationInPage »HFPDFLink_GetLinkAtPoint ¼HFPDFLink_GetLinkZOrderAtPoint ½HFPDFLink_GetDest ¾HFPDFLink_GetAction ¿HFPDFLink_Enumerate ÀHFPDFLink_GetAnnot ÁHFPDFLink_GetAnnotRect ÂHFPDFLink_CountQuadPoints ÃHFPDFLink_GetQuadPoints ÄHFPDF_GetPageAAction ÅHFPDF_GetFileIdentifier ÆHFPDF_GetMetaText ÇHFPDF_GetPageLabel ÈHEPDF_SetMetaText ÉHEPDF_HasMetaText ÊHEPDF_GetMetaTrapped ËHEPDF_SetMetaTrapped ÌHEPDF_GetMetaKeyCount ÍHEPDF_GetMetaKeyName ÏHFPDFPageObj_NewImageObj ²EFPDFImageObj_LoadJpegFile ³EFPDFImageObj_LoadJpegFileInline µEFPDFImageObj_SetMatrix ¶EFPDFImageObj_SetBitmap ·EFPDFImageObj_GetBitmap ¸EFPDFImageObj_GetRenderedBitmap ¹E FPDFImageObj_GetImageDataDecoded ºEFPDFImageObj_GetImageDataRaw »E FPDFImageObj_GetImageFilterCount ¼EFPDFImageObj_GetImageFilter ½EFPDFImageObj_GetImageMetadata ¾EFPDFImageObj_GetImagePixelSize ¿E%FPDFImageObj_GetIccProfileDataDecoded ÀEFPDF_CreateNewDocument öEFPDFPage_Delete øEFPDF_MovePages ùEFPDFPage_New úEFPDFPage_GetRotation ûEFPDFPage_InsertObject ýEFPDFPage_InsertObjectAtIndex ÿEFPDFPage_RemoveObject €FFPDFPage_CountObjects FFPDFPage_GetObject ‚FFPDFPage_HasTransparency ƒFFPDFPageObj_Destroy „FFPDFPageObj_GetMarkedContentID …FFPDFPageObj_CountMarks †FFPDFPageObj_GetMark ‡FFPDFPageObj_AddMark ˆFFPDFPageObj_RemoveMark ‰FFPDFPageObjMark_GetName ŠFFPDFPageObjMark_CountParams ‹FFPDFPageObjMark_GetParamKey ŒF!FPDFPageObjMark_GetParamValueType F FPDFPageObjMark_GetParamIntValue ŽF#FPDFPageObjMark_GetParamStringValue F!FPDFPageObjMark_GetParamBlobValue FFPDFPageObj_HasTransparency ‘FFPDFPageObjMark_SetIntParam ’FFPDFPageObjMark_SetStringParam “FFPDFPageObjMark_SetBlobParam ”FFPDFPageObjMark_RemoveParam –FFPDFPageObj_GetType —FFPDFPageObj_GetIsActive ˜FFPDFPageObj_SetIsActive ™FFPDFPage_GenerateContent šFFPDFPageObj_Transform ›FFPDFPageObj_TransformF œFFPDFPageObj_GetMatrix FFPDFPageObj_SetMatrix žFFPDFPageObj_SetBlendMode ŸFFPDFPage_TransformAnnots  FFPDFPage_SetRotation ¡FFPDFPageObj_SetFillColor ¢FFPDFPageObj_GetFillColor £FFPDFPageObj_GetBounds ¤FFPDFPageObj_GetRotatedBounds ¥FFPDFPageObj_SetStrokeColor ¦FFPDFPageObj_GetStrokeColor §FFPDFPageObj_SetStrokeWidth ¨FFPDFPageObj_GetStrokeWidth ©FFPDFPageObj_GetLineJoin ªFFPDFPageObj_SetLineJoin «FFPDFPageObj_GetLineCap ¬FFPDFPageObj_SetLineCap ­FFPDFPageObj_GetDashPhase ®FFPDFPageObj_SetDashPhase ¯FFPDFPageObj_GetDashCount °FFPDFPageObj_GetDashArray ±FFPDFPageObj_SetDashArray ²FFPDFFormObj_CountObjects ´FFPDFFormObj_GetObject µFFPDFFormObj_RemoveObject ¶FFPDFPageObj_CreateNewPath ÛIFPDFPageObj_CreateNewRect ÜIFPDFPath_CountSegments ÝIFPDFPath_GetPathSegment ÞIFPDFPath_MoveTo ßIFPDFPath_LineTo àIFPDFPath_BezierTo áIFPDFPath_Close âIFPDFPath_SetDrawMode ãIFPDFPath_GetDrawMode äIFPDFPathSegment_GetPoint åIFPDFPathSegment_GetType æIFPDFPathSegment_GetClose çIFPDFPageObj_NewTextObj ßGFPDFText_SetText àGFPDFText_SetCharcodes áGFPDFText_LoadFont âGFPDFText_LoadStandardFont íGFPDFText_LoadCidType2Font îGFPDFTextObj_GetFontSize ïGFPDFTextObj_GetText ðGFPDFTextObj_GetRenderedBitmap ñGFPDFFont_Close òGFPDFPageObj_CreateTextObj óGFPDFTextObj_GetTextRenderMode ôGFPDFTextObj_SetTextRenderMode õGFPDFTextObj_GetFont öGFPDFFont_GetBaseFontName ÷GFPDFFont_GetFamilyName øGFPDFFont_GetFontData ùGFPDFFont_GetIsEmbedded úGFPDFFont_GetFlags ûGFPDFFont_GetWeight üGFPDFFont_GetItalicAngle ýGFPDFFont_GetAscent þGFPDFFont_GetDescent ÿGFPDFFont_GetGlyphWidth €HFPDFFont_GetGlyphPath H FPDFGlyphPath_CountGlyphSegments ‚H!FPDFGlyphPath_GetGlyphPathSegment ƒHEPDFText_RedactInRect „HEPDFText_RedactInQuads …HFPDFDoc_GetPageMode ›KFPDFPage_Flatten ‘KFPDFPage_HasFormFieldAtPoint þFPDFPage_FormFieldZOrderAtPoint ÿmalloc ª8free ¬8FORM_OnMouseMove ‚FORM_OnMouseWheel ƒFORM_OnFocus „FORM_OnLButtonDown …FORM_OnLButtonUp †FORM_OnLButtonDoubleClick ‡FORM_OnRButtonDown ˆFORM_OnRButtonUp ‰FORM_OnKeyDown ŠFORM_OnKeyUp ‹FORM_OnChar ŒFORM_GetFocusedText FORM_GetSelectedText ŽFORM_ReplaceAndKeepSelection FORM_ReplaceSelection FORM_SelectAllText ‘FORM_CanUndo ’FORM_CanRedo “	FORM_Undo ”	FORM_Redo •FORM_ForceToKillFocus –FORM_GetFocusedAnnot —FORM_SetFocusedAnnot ˜FPDF_FFLDraw šFPDF_SetFormFieldHighlightColor ›FPDF_SetFormFieldHighlightAlpha œFPDF_RemoveFormFieldHighlight FORM_OnAfterLoadPage žFORM_OnBeforeClosePage ŸFORM_DoDocumentJSAction  FORM_DoDocumentOpenAction ¡FORM_DoDocumentAAction ¢FORM_DoPageAAction £FORM_SetIndexSelected ¤FORM_IsIndexSelected ¥ FPDFDoc_GetJavaScriptActionCount ÕIFPDFDoc_GetJavaScriptAction ÖIFPDFDoc_CloseJavaScriptAction ØIFPDFJavaScriptAction_GetName ÙIFPDFJavaScriptAction_GetScript ÚIFPDF_ImportPagesByIndex ¦IFPDF_ImportPages §IFPDF_ImportNPagesToOne ¨IFPDF_NewXObjectFromPage ©IFPDF_CloseXObject ªIFPDF_NewFormObjectFromXObject «IFPDF_CopyViewerPreferences ¬I*FPDF_RenderPageBitmapWithColorScheme_Start KFPDF_RenderPageBitmap_Start ŽKFPDF_RenderPage_Continue KFPDF_RenderPage_Close KFPDF_SaveWithVersion °"FPDFText_GetCharIndexFromTextIndex ìK"FPDFText_GetTextIndexFromCharIndex íKFPDF_GetSignatureCount ÿJFPDF_GetSignatureObject ‚KFPDFSignatureObj_GetContents ƒKFPDFSignatureObj_GetByteRange „KFPDFSignatureObj_GetSubFilter …KFPDFSignatureObj_GetReason †KFPDFSignatureObj_GetTime ‡K$FPDFSignatureObj_GetDocMDPPermission ˆKFPDF_StructTree_GetForPage îHFPDF_StructTree_Close ïHFPDF_StructTree_CountChildren ðHFPDF_StructTree_GetChildAtIndex ñHFPDF_StructElement_GetAltText òH FPDF_StructElement_GetActualText óHFPDF_StructElement_GetID ôHFPDF_StructElement_GetLang õH$FPDF_StructElement_GetAttributeCount öH&FPDF_StructElement_GetAttributeAtIndex ÷H%FPDF_StructElement_GetStringAttribute øH%FPDF_StructElement_GetMarkedContentID ùHFPDF_StructElement_GetType úHFPDF_StructElement_GetObjType ûHFPDF_StructElement_GetTitle üH FPDF_StructElement_CountChildren ýH"FPDF_StructElement_GetChildAtIndex þH*FPDF_StructElement_GetChildMarkedContentID ÿHFPDF_StructElement_GetParent €I FPDF_StructElement_Attr_GetCount IFPDF_StructElement_Attr_GetName ‚I FPDF_StructElement_Attr_GetValue ƒIFPDF_StructElement_Attr_GetType „I'FPDF_StructElement_Attr_GetBooleanValue …I&FPDF_StructElement_Attr_GetNumberValue †I&FPDF_StructElement_Attr_GetStringValue ‡I$FPDF_StructElement_Attr_GetBlobValue ˆI%FPDF_StructElement_Attr_CountChildren ‰I'FPDF_StructElement_Attr_GetChildAtIndex ŠI*FPDF_StructElement_GetMarkedContentIdCount ‹I,FPDF_StructElement_GetMarkedContentIdAtIndex ŒIFPDF_AddInstalledFont ãJFPDF_SetSystemFontInfo äJFPDF_GetDefaultTTFMap åJFPDF_GetDefaultTTFMapCount æJFPDF_GetDefaultTTFMapEntry çJFPDF_GetDefaultSystemFontInfo éJFPDF_FreeDefaultSystemFontInfo ñJFPDFText_LoadPage ŽEFPDFText_ClosePage EFPDFText_CountChars EFPDFText_GetUnicode ‘EFPDFText_GetTextObject ’EFPDFText_IsGenerated “EFPDFText_IsHyphen ”EFPDFText_HasUnicodeMapError •EFPDFText_GetFontSize –EFPDFText_GetFontInfo —EFPDFText_GetFontWeight ˜EFPDFText_GetFillColor ™EFPDFText_GetStrokeColor šEFPDFText_GetCharAngle ›EFPDFText_GetCharBox œEFPDFText_GetLooseCharBox EFPDFText_GetMatrix žEFPDFText_GetCharOrigin ŸEFPDFText_GetCharIndexAtPos  EFPDFText_GetText ¡EFPDFText_CountRects ¢EFPDFText_GetRect £EFPDFText_GetBoundedText ¤EFPDFText_FindStart ¥EFPDFText_FindNext ¦EFPDFText_FindPrev §EFPDFText_GetSchResultIndex ¨EFPDFText_GetSchCount ©EFPDFText_FindClose ªEFPDFLink_LoadWebLinks «EFPDFLink_CountWebLinks ¬EFPDFLink_GetURL ­EFPDFLink_CountRects ®EFPDFLink_GetRect ¯EFPDFLink_GetTextRange °EFPDFLink_CloseWebLinks ±E FPDFPage_GetDecodedThumbnailData ”KFPDFPage_GetRawThumbnailData –KFPDFPage_GetThumbnailAsBitmap —KFPDFPage_SetMediaBox ÀIFPDFPage_SetCropBox ÁIFPDFPage_SetBleedBox ÂIFPDFPage_SetTrimBox ÃIFPDFPage_SetArtBox ÄIFPDFPage_GetMediaBox ÅIFPDFPage_GetCropBox ÇIFPDFPage_GetBleedBox ÈIFPDFPage_GetTrimBox ÉIFPDFPage_GetArtBox ÊIFPDFPage_TransFormWithClip ËIFPDFPageObj_TransformClipPath ÍIFPDFPageObj_GetClipPath ÎIFPDFClipPath_CountPaths ÏIFPDFClipPath_CountPathSegments ÐIFPDFClipPath_GetPathSegment ÑIFPDF_CreateClipPath ÒIFPDF_DestroyClipPath ÓIFPDFPage_InsertClipPath ÔIFPDF_InitLibrary ÎFPDF_DestroyLibrary ÐFPDF_SetSandBoxPolicy ÑFPDF_LoadDocument ÒFPDF_GetFormType ÔFPDF_LoadXFA ÕFPDF_LoadMemDocument ÖFPDF_LoadMemDocument64 ×FPDF_LoadCustomDocument ØFPDF_GetFileVersion Ù(FPDF_DocumentHasValidCrossReferenceTable ÚFPDF_GetDocPermissions ÛFPDF_GetDocUserPermissions ÜFPDF_GetSecurityHandlerRevision ÝFPDF_GetPageCount Þ
FPDF_LoadPage ßFPDF_GetPageWidthF àFPDF_GetPageWidth áFPDF_GetPageHeightF âFPDF_GetPageHeight ãFPDF_GetPageBoundingBox äFPDF_RenderPageBitmap åFPDF_RenderPageBitmapWithMatrix æEPDF_RenderAnnotBitmap çFPDF_ClosePage èFPDF_CloseDocument éFPDF_GetLastError êFPDF_DeviceToPage ëFPDF_PageToDevice ìFPDFBitmap_Create íFPDFBitmap_CreateEx îFPDFBitmap_GetFormat ïFPDFBitmap_FillRect ðFPDFBitmap_GetBuffer ñFPDFBitmap_GetWidth òFPDFBitmap_GetHeight óFPDFBitmap_GetStride ôFPDFBitmap_Destroy õFPDF_GetPageSizeByIndexF öEPDF_GetPageRotationByIndex ÷FPDF_GetPageSizeByIndex øFPDF_VIEWERREF_GetPrintScaling ùFPDF_VIEWERREF_GetNumCopies ú FPDF_VIEWERREF_GetPrintPageRange û%FPDF_VIEWERREF_GetPrintPageRangeCount ü'FPDF_VIEWERREF_GetPrintPageRangeElement ýFPDF_VIEWERREF_GetDuplex þFPDF_VIEWERREF_GetName ÿFPDF_CountNamedDests €FPDF_GetNamedDestByName FPDF_GetNamedDest ‚FPDF_GetXFAPacketCount ƒFPDF_GetXFAPacketName †FPDF_GetXFAPacketContent ‡FPDF_GetTrailerEnds ˆfflush ê6emscripten_builtin_memalign ¯8strerror ”8setThrew ´8_emscripten_tempret_set ¸8emscripten_stack_init ûJemscripten_stack_get_free üJemscripten_stack_get_base ýJemscripten_stack_get_end þJ_emscripten_stack_restore àJ_emscripten_stack_alloc áJemscripten_stack_get_current âJ
dynCall_ji üKdynCall_jij ýKdynCall_iiij þKdynCall_iij ÿK	dynCall_j €LdynCall_jji LdynCall_iji ‚LdynCall_viijii ƒLdynCall_iiji „LdynCall_jiji …LdynCall_iiiiij †LdynCall_iiiiijj ‡LdynCall_iiiiiijj ˆLdynCall_viji ‰L	Ô/ Aç'‰Œ‘¤¦§¥¨©ª«¬­®¯·¸¹º»¼½¾¿ÀÁÂÃÄÆÇÈÎÉÝÊËãäæèêìîÍòôÒÏÓÑÌ—˜™›ÜÞßšð¿œöØâËÌDÓÔÕÖ×ÝÞãäåæçèéêë®¯ÄÅÉØÎÙÚÛÜÝÞßàáâãäåæôõö÷üøùûúýÛ°±ŒŽ‘š›’ž ¡²³ÄÅÆÇÈËÉÊÌØÙÚÛÜßÝÞàãåæèç”éú¸$âãäÓåéêüýþÿžµ¶™›šœ¢£¤¥¦§¨©ªÔ°¬­®¯¶·²´³µ½¾¹»º¼âãäåæçôõÅÆÈÊËÌûüþÿù,â,û,ä,âçéèæåäãíîïðñ¶	·	¹	º	À	Á	Ä	Å	Æ	ð	ñ	ò	ó	…
†
‡
‰
›
œ

ž
 
¡
½
¿
Á
É
Ê
Ë
Í
Î
Ì
Ñ
Ò
Ó
Ô
Õ
Ö
ñ
õ
‚ƒ„’ßàäâôõøù¢£¤‹ŒŽ’‡–™¥¦ˆ§¨©ª«¬­žŸ·¸ÃÄÆÞÇÙÚÜÝßž
Ÿ
¹
º
Ã
Ä
È
ˆ‰Š‹É
Ê
Ž‘’“”•è
é
î
ï
ñ
ÚÛò
ÝÞßàð
ô
õ
Üö
ø
ù
ü
ý
û
€‡‚ƒš›œ ¤¥¦§©ª«°¿ÀÁÂÈÉîâäåæçêëìíïðñòóôõö÷øùúûüýþÿ€‚ƒ„˜™š°±àáèé‚ƒ…†‡Œ ¢£¤¦§¨³µ§º½¿Á´¶»¾ÀÂÛÜÝÞáâãêëìíïðñ„…†‡‰Š‹Ž•’“”™›œžŸ¢¦§©«¬°±³´µ¶·¸¹º»¼½¾ÀÃÄÆÈÉÊËšÍÏÐÑÒÓÔÕ×ØÙÚÛÜÝÞßàáâãäåæçêëíïðñòóô÷øù‰Š‹›œž¦ÓÖØÙÚÛÜâáäÝÕ×ÞðÔñò¥¦¨©ª«¬­°±²³´µ¶ÏÐÑÒÓÔÕÖ×ØÙÚ™š›œŸ ¡¥­®ÈÉÊÝÞãäîï„…†‰Š‹Ž—™­¯¨±–˜²¼½¾¿ÀÁÂÃÄýþ€‚ƒ„­®¯°±²³¹ºÉÊËÌ¡ÚÛàáâãøäêéùúû–—˜™š›œžŸ õìëüþ€‚„†ˆŠŒŽ’”ýÿƒ…‡‰‹‘“•ª¬­®¯³´Üßàâä·ê„†‡¢£¤¥¦§¨©ª«¬¶é…ˆŠŒŽ’”–˜šœž ‰‹‘“•—™›Ÿ¡¼ÀÐöøŠ‹ŒŽ”•–—Œ9Ž999˜™ß8à8š›ä8å8æ8œï8ò8ž¦«º»­°²¸¾ÀÃ®ÇÉÌÅÙß¼ª¬¿ÂÄ·ÊËÍØÞ¯±³¹à½ÁëìÆíîýðñòóôõö÷øùúûüþÿ€‚ƒ…†Œ‘’“— ¡¢£¤¦§¨©­¯°±²³òô™¦§­©«ª¬µ·¸¹ÁÂ½¿¾ÀŒ¹º–—˜™šžŸ ¡£¤¥§¨©Ç¿À¸ÁÂ¼½ÅÆ‘ÈÊÌÓÔÎÏÐÑÒËÍÖØÞÛÜÚßà×ÙÝâãáäåçé÷íõöñòëìóôîïøúèêðøùûý¤¥¦§¨­¯°²´¸ÀÁ½¶·¾¿¹º³µ¼ÂÃ»ÄÅÇÈÉËÍÏÌÎÑÓØÖ×ÕÙÚÒÔÛÜÝßáîåêëãäìíæçàâéïðèñòôöõ÷ùüýÿ€‚ƒ„…†‡ˆ‰Š‹ŒŽ‘’“”•–˜™š›œž¼¶·¾ ž Ÿ   ¡ ¢ £ ¤ ¥ ¦ ² ³ µ ¶ · ¾ Ô Õ ¹ º » ¼ ½ ¿ À Á Â Ã Ä Å Ë Î Ð Ñ Ò Ó × Ø Ù Ú Û Ü Ý Þ ß à á â ã úûä å ç è é í î ï ð ñ ò ó ô Š!þ ÿ €!!„!…!†!‡!ˆ!‰!‹!Œ!!Ž!!!‘!“!”!•!–!½!¾!¿!À!É!Þ!—!˜!™!š!›!œ!!Ÿ!£!¥!§!©!ª!«!¬!­!®!¯!°!±!²!³!´!µ!¶!·!¸!¹!º!»!¼!Ã!Ä!Å!Ô!Õ!Ö!×!Ø!Ù!Ú!Û!Ü!Ý!ß!à!á!Ê!Ë!Ì!Í!Î!Ï!Ð!Ñ!Ò!Ó!â!ã!ä!å!†"‡"ˆ"‰"Š"‹"Œ"""Ž"""‘"’"“"–"—"˜"…"”" "ž"Ÿ"¡"¢"£"¤""°"±"²"³"µ"¶"·"¸"¿"¥"¦"§"¨"ª"«"¬"­"®"¯"ú"˜#™#Àáþ"ÿ"€#ƒ#Á"„#…#†#‡#ˆ#‰#Š#‹#Œ##Ž###“#”#•#–#—##š#Â"Ã"Ä"Å"Æ"Ç"È"É"Ê"Ë"Ì"Ï"Ñ"Ò"Ó"Ô"Õ"Ö"×"Ø"Ù"Ú"Û"Ü"Ý"Þ"ß"à"â"ä"å"æ"ç"é"ë"ì"í"î"ï"ð"ñ"ò"ó"ô"õ"÷"ø"ù"û"ü"ý"›#œ##ž#Ÿ# #¡#¹#£#ò¤#¥#¦#§#¨#ª#«#¬#°#±#²#´#µ#·#¸#Å#Æ#Ç#È#É#þ#Ð#Ö#×#Ø#Ù#Ú#Û#Ü#Ý#Þ#ß#à#æ#ç#è#é#ê#ë#í#õ#ö#÷#ø#ù#ú#û#ü#º#»#¼#½#Ê#Ë#Ì#Î#Ï#Ó#Ô#Õ#ï#ð#ñ#†$$¨$ÿ#€$$‚$ˆ$‰$Š$‹$Œ$$Ž$$’$“$”$•$–$—$˜$™$š$›$œ$$ž$Ÿ$ $¢$£$¤$¥$¦$§$…$©$ª$«$¬$­$®$°$±$²$³$¾$Á$Å$È$š%›%ž%%œ%Ÿ% %£%¤%¥%¦%±%²%³%´%µ%·%¸%¹%º%»%«%¬%­%®%¯%é%ê%ë%ì%í%î%ï%ð%ñ%ò%ó%ô%õ%ö%÷%ø%ù%ú%û%ü%ý%þ%ÿ%€&&‚&ƒ&„&…&†&‡&ˆ&‰&Š&¢&¬&±&²&³&·&¸&¹&»&¼&½&À&Â&Ç&Ï&Ñ&Ù&Ú&÷&ø&ù&ú&•'–'—'˜'™'š'›'œ''ž'Ÿ' '¡'¢'£'¤'¥'¦'§'¨'©'ª'«'¬'­'®'¯'°'±'²'³'´'µ'¶'·'¸'¹'º'»'¼'½'¾'¿'À'Á'Â'Ã'Ä'Å'Æ'Ç'È'É'Ê'Ë'Ì'Í'Î'Ï'Ð'Ñ'Ò'Ó'Ô'Õ'Ö'×'Ø'Ù'Ú'Û'Ü'Ý'Þ'ß'à'á'â'ã'ä'å'æ'ç'è'é'ê'ë'ì'í'î'ï'ð'ñ'ò'ó'ô'õ'ö'÷'ø'ù'ú'û'ü'ý'þ'ÿ'€((‚(ƒ(„(…(†(‡(ˆ(‰(Š(‹((‘(™(›(“(˜(š(Ÿ(¡(¢(£(¥(¦(¨(«(¬(­(®(¶(·(¸(³(´(µ(©(¯(±(ž(Æ)Ø)Ú)î)ð)ß(à(á(â(ã(ä(å(æ(ç(è(é(ê(ë(ì(í(î(ï(ð(ñ(ò(ó(ô(õ(ö(÷(ø(ù(ú(û(ü(ý(þ(ÿ(€))‚)ƒ)…)‡)ˆ)‰)‹))Ž)))‘)’)“)”)•)–)—)™)š)›)œ)ž)Ÿ) )¡)¢)£)¤)¥)¦)§)¨)©)ª)«)¬)­)±)µ)¶)·)¸)¹)º)»)¼)½)¾)¿)Á)Ã)Ä)Å)È)É)Ê)Ë)Ì)Í)Î)Ï)Ð)Ñ)Ò)Ó)Ô)Õ)Ö)×)Ù)Ü)Ý)Þ)â)ä)å)æ)ç)Ü(è)é)ê)ë)ì)í)ï)ñ)ò)ó)ô)õ)ö)÷)ø)ù)ú)û)ü)Œ**Ž***‘*’*“*”*•*–*—*˜*™*š*›*œ**ž*Ÿ* *¡*¢*£*¤*¥*¦*§*¨*©*ª*«*¬*³*´*µ*¶*·*¸*¹*º*»*¼*Ä*Å*Æ*Ç*È*É*Ê*Ë*Ú*ê*ì*ñ*ò*þ*ÿ*€+++¸+“+—+˜+›+œ++Ÿ+ +¡+£+¤+¥+¦+§+¨+©+ª+«+¬+­+®+¯+°+±+²+³+´+µ+¶+·+Á+Î+Ï+Ð+Ñ+Ò+Ó+Ô+Õ+Ö+Ž++‰+ü*+ö*õ*ù*ý*÷*…+”+’+†+„+‚+Ë+Ì+Ê+À+»+¿+Æ+Í+È+Ç+¾+½+Å+Ä+Ã+Â+ˆ,‰,,ž,Ø,Ù,É,Ì,Î,æ,ç,è,é,ê,ë,ì,í,î,ï,þ,ÿ,€--„-…-†-‡-ˆ-Œ--Ž--‘-’-“-”-–-—-™-š-›-œ--¡-¢-£-¤-¥-¦-§-¨-©-«-¬-®-°-±-²-³-´-µ-¶-·-¸-¹-º-¼-½-¾-¿-Á-Â-Ã-Ä-Å-Æ-Ç-È-É-Ê-Ë-Ñ-Ò-Ô-Õ-Ö-Ø-Ù-Ú-Û-Ü-Ý-ñ-à-ß-å-Þ-ä-ã-â-æ-ç-è-é-ê-ë-ì-í-á-î-ï-ó-ô-õ-ö-÷-ø-ù-û-ü-ý-þ-‚.ƒ.….†.‡.ˆ.‰.‹.Œ..Ž...‘.’.“.”.•.Ÿ. .¢.¤.¥.¦.§.¨.©.ª.«.¬.­.®.°.±.².³.´.µ.·.¸.¹.º.».¼.½.¾.¿.À.Á.Ã.Ä.Å.Æ.È.É.Ê.Ë.Í.Î.Ï.Ð.Ñ.Ò.Ó.Ô.Õ.Ö.Ù.Ú.Û.Ü.Ý.Þ.ß.á.ý.ü.ò.û.ñ.ð.ï.ó.ô.õ.ö.÷.ø.ù.ú.î.í.ì.ã.ä.å.æ.ç.è.é.ê.ë.ÿ.€//ƒ/„/…/†/‡/‰/Š/‹/Œ//Ž//“/”/•/–/—/˜/™/š/›/œ//ž/Ÿ/ /¢/£/¤/¥/¦/§/©/ª/«/¬/­/®/¯/°/±/²/³/µ/¶/·/¸/º/¼/½/¾/¿/À/Á/Â/Ã/Ä/Å/Ú/Û/Ü/Ý/Þ/ß/á/â/ã/ä/å/æ/ì/í/¥0¦0§0î/©0ª0«0¬0­0®0ï/ð/²0ò/‰0…0‡0ˆ0þ/ó/õ/·0‚0ý/÷/Œ00£0¤0¨0¯0°0³0½0±0´0µ0¶0ð0ñ0ò0ó0ô0õ0÷0ø0ƒ1ù0„1…1†1ÿ0€1ˆ1‰1Š1‹1Œ11Ž1’1“1”1•1–1—1˜1¤1¦1§1¨1©1ª1±1²1«1¬1¶1·1¹1»1¼1Ä1Ç1Í1Ï1Ð1Î1Ñ1Ò1Ô1Õ1´2¶2ä2å2æ2ç2é2ê2ƒ6…6†6‡6ì2í2‹6Œ6Š6–6˜6™6š6›6œ66¨6ª6³6®6¸6·66Ž666‘6’6“6”6•6¶6ž6«6¬6²6±6ë2ò5ó5ï2ñ2ä4æ4ç4ó2é4„5ê4ë4ì4ñ4ò4ð4î4í4ã4á4â4ó4õ4÷4ð2ò2ô4ö4ø4ù2ú2“3•3ˆ6‰6–3—3ü2ý2‡3ÿ2€33‚3ƒ3„3…3†3’3Œ33ü5û2‘3˜3™3›33¡3¤3Ÿ3 3œ3ž3¢3¦3§3º3¾3Ô3Ò3Ó3Õ3Ö3Å3Í3Î3Ï3È3¯3Æ3×3É3Ê3Ë3Ì3Ù3Ú3Á3À3¹3ª3±3´3Ð3©3°3­3Ç3á3â3ã3ä3ë3ì3ó3í3ù3ú3ý3û3þ3ÿ34€4‚4ƒ4…4„4†4‡4Š4ˆ4‹4Œ4‘44”4•4—4–4Ý4ß4è4Þ4à4­5®5¯5°5²5³5´5¶5·5¹5¸5»5¼5¾5½5À5Á5Ã5Ä5Å5Â5Ç5É5Ñ5Ò5Ó5Õ5Ö5Ù5Ú5Þ5Ì5Ð5Ë5ß5È5Ê5à5Ï5á5â5ì5í5»6¼6½6æ5¾6¿6í6î6ï6ð6×7å7æ7‹8Œ88Ÿ8Û8Ü8á8ã8í8ñ8ó8É:Ê:÷:¬8Ñ=è?þ?†@“@A„AˆA‹AŽA‘A“A•A—A™A›AAŸA¡Aó?÷?@¤@¥@¦@§@¨@©@ª@«@¬@­@ø>·@¸@»@¾@¿@Â@Ã@Å@ì@í@ð@ò@ô@ö@ú@î@ï@ñ@ó@õ@÷@û@š;Ž@”@•@–@—@˜@™@›@œ@ž@Ÿ@ @¡@¢@®@¯@°@±@²@³@´@µ@Æ@Ç@É@Ë@Ì@Í@Î@Ð@Ñ@Ò@Ó@Ô@Õ@Ö@×@Ø@Ù@Ú@Ü@Þ@ß@à@á@ã@ä@å@æ@ç@è@é@ê@ë@™;›;œ;; ;¡;¢;£;¤;¨;¤A©;·;À;Ã;Æ;É;Ì;Ï;Ô;×;Ú;¥Aá;ë;ð;ò;ô;ö;ø;ú;þ;€<‚<¦A“<›<¡<£<¥<§<°<²<§A¶<¿<Ã<Å<Ç<É<Ï<Ñ<¨AªAÚ<Û<Ü<Ý<ß<á<ä<ÿ@†AŒAšAžA’A–A«A­Aó<ô<õ<û<ý<ÿ<‚=‚A‰AAœA A”A˜A¯A®A=±A°A•=²A›=ž=Ÿ= =¡=¢=£=¤=¥=³A¦=§=¨=©=ª=«=¬=­=®=´A¯=²=³=´=¸=¹=º=»=¼=µA½=¾=¿=À=Á=Â=Ã=Ä=Å=¶AÐ=è=·A>¢>¸AÐ>Ü>¹AÝ>ê>ºAò>ó>ô>»Aõ>ö>÷>ÚCÛCŽD‘DDD–D«D¨DD’DªD§DžD“D©D¤D¡D³D­DïDéDêDëDìDíDîDðDñDòDóDôDõDÊEËEÐEÍEÎEÏE›HHÔHÖHéIëIóIøIùIúIŽJJœJ¢J£JÐJÖJ×J™JØJÙJÚJÛJÜJÝJÞJßJêJëJìJíJèJîJïJðJòJóJôJõJöJ÷JøJùJúJŠK‹KŒK¢KÃK·8®K¡K«KªKKÇKÅKäK°KèKÁKÈKëK£KéKâKæKãK
×Ç©êK ûJŒ7Ì7ÙCo
# !A!  k! $    6 (!A!  j! &A!  6 A!  6A!	  	j!
 
$  €
# !A!  k! $    6 (!A!  j! !A
!  j!	 	!
   
(A !  )A!  j!
 
$  “# !A!  k! $    6  6  6 (!  6  ( !A!  j!	 (!
 (! 	 
 õC (!A!
  
j! $  Y# !A!  k! $    6  6  6 (! + ,A!  j! $  &# !A!  k!   6  6V# !A!  k! $    6 (!A!  j! îCA!  j! $  S~# !A!  k!   6 (!B !  7 A!  j!A !  6  F# !A!  k! $    6 (! -A!  j! $  F# !A!  k! $    6 (! .A!  j! $  *# !A!  k!   6 (! Š
# ! A !   k! $ A!  6A !  6A !  6A !  6A !  6A!  j!	 	!
 
ÏA !  j! $ A!   ÝC! % X	# !A!  k! $    6 (!A!  j! 2!A!  j!	 	$  I# !A!  k! $    6 (! 3!A!  j! $  …
# !A!  k! $    6 (! =!A!  q!@@ E
  >! !	 ?!
 
!	 	!A!  j!
 
$  ‹# !A!  k! $    6  6  6 (! (!A!  j!	 	5!
 (!  
 Ö6 (!A!
  
j! $  I# !A!  k! $    6 (! 6!A!  j! $  Q# !A!  k! $    6 (! B! C!A!  j! $  n
# !A!  k! $    6 (!A !  F!A!  q!@ 
  * áCA!	  	j!
 
$ t
# !A!  k! $    6  6 (!  6 (! (!A !   ®!	A!
  
j! $  	Ê# ! A!   k! $ AŒ! ÝC!AŒ!A !   Ø6  6 (!AŒ!A !	  	 Ø6 (!
A! 
 6  (!A !
  
6 (!A !  6@ (!A!  j! $  i
# !A!  k! $    6 (!A !  F!A!  q!@ 
  áCA!	  	j!
 
$ \# !A!  k! $    6  6 (! (!  €!A!  j!	 	$  D# !A!  k! $    6 (! A!  j! $ Ÿ# !A!  k! $    6 (! @! - !A!  v!A !	Aÿ!
  
q!Aÿ! 	 q!
  
G!A!  q!A!  j! $  R# !A!  k! $    6 (! @! (!A!  j! $  r# !A!  k! $    6 (! @! - !Aÿ !  q!Aÿ!	  	q!
A!  j! $  
I# !A!  k! $    6 (! A!A!  j! $  *# !A!  k!   6 (! …
# !A!  k! $    6 (! =!A!  q!@@ E
  D! !	 E!
 
!	 	!A!  j!
 
$  *# !A!  k!   6 (! R# !A!  k! $    6 (! @! ( !A!  j! $  Q# !A!  k! $    6 (! @! F!A!  j! $  *# !A!  k!   6 (! :~A !@ ­  ­~"B ˆ§" 
 A  §  " AþßÿÿK
   ª8! 3~A !@ ­  ­~"B ˆ§
  §AþßÿÿK
    ³8! <~A !@ ­ ­~"B ˆ§"
 A  § "AþßÿÿK
    ­8!    ¬8&~A !@ ­  ­~"B ˆ§
  §ª8!    ¬8   # Ak  6   AG    H
    AI   J
 A°µë OÅ6 ;# Ak"$  A 6@@ Aj   ±8
  (" 
  Aj$    @   G"
  T   @   H"
  T  - @@  A nO
    l H"E
  T  T  @    I"
  T   @   K"
  T  C@  AdO
   AjApq"AZ"   Asja@ E
   ( Aj6   —~@  ("AdO
  AjApq"AZ"  Asja@ E
   ( Aj6  (Aj"  ) "B ˆ§" I
  Aj!@ B€€€€T
   §  ×6   M
    jA :    L@  (Aj" ("I
   Aj! @ E
    (  ×6  O
    jA :   !    ( Aj"6 @ A J
   LC@ (Aj" (AjK
   (Aj I
 @ E
   Aj Aj ×6 j@  (Aj" I
   k ("I
   Aj!@ E
   j (  ×6  (Aj!  j" O
   jA :   4    6   6  A 6 @  AjI
     jAjA :    W@  AùÿÿÿO
   At"Aa AaIAjApq"AZ"   ApjAvh@ E
   ( Aj6   ¨~@  ("AùÿÿÿO
  AtAjApq"AZ"  ApjAvh@ E
   ( Aj6  (Aj"  ) "B ˆ§" I
  Aj!@  AÿÿÿÿqE
   §  At×6   M
    AtjA 6   Y@  (Aj" ("I
   Aj! @ AÿÿÿÿqE
    (  At×6  O
    AtjA 6  !    ( Aj"6 @ A J
   LM@ (Aj" (AjK
   (Aj I
 @ AÿÿÿÿqE
   Aj Aj At×6 z@  (Aj" I
   k ("I
   Aj!@ AÿÿÿÿqE
   Atj (  At×6  (Aj!  j" O
   AtjA 6  7    6   6  A 6 @  AjI
     AtjAjA 6   1@  ( "E
 @ ( AJ
  A 6  A 6  ^‹@ ( "
 @ 
   B 7  [! ( !  6 @ E
  ^ ( ! A 6 ( A :    ( "(6   Aj6 @ ( AJ
   ("K
    6   Aj6 @  ("  K"
   B 7  [" ( _  ( (6 ( !  6 @ E
  ^ ( !   (6   Aj6 ž@@  ( "E
 @ ("   I"
 @ ( AJ
  A 6  A 6  ^  6   ( "(AjO
  jAjA :    ( "( kA I
   ( Aj6    l ^ ù# Ak"$ @@@@  ( "E
 @ ( "AJ
   (M
 
@ AJ
  A 6  A 6  ^ E
 [!@@  ( "
 A ! (!  Aj6     I"6  )7   ]  6  (AjO
  jAjA :    ( !   6  E
  ^ Aj$  Í@@  ( "
 A ! (!@@  K
    Aj"l Aj"  ( "("Aj"K
  I
  k"  kK
  k I
@  F
  Aj" j  j ×6  ( "(Aj!  O
  jAj :    (  6  ¢A !@@  ( "E
  ("E
  Aj! Aq!A !@@ AO
 A ! A|q!A ! Aÿq!A !	@  -   Fj Aj-   Fj Aj-   Fj Aj-   Fj! Aj! 	Aj"	 G
 @ E
  Aÿq!@  -   Fj! Aj! Aj" G
 @ 
 A    (l@@  ( "("
 A ! Aÿq! ! Aj"!	@@ -  " F
  E
 	 :   	Aj!	 Aj! Aj! Aj"
   ( "(!   k6  ( "(" (AjO
  jAjA :    ¹@  ( "
 A  (!@@ E
   K
    j"I
    l  ( "(Aj" I
  I
  kAj"  kK
  k I
@ E
  Aj" j  j ×6  ( !   k6  ( (!  ²~@ ( "E
  (" M
  ) "B€€€€T
   k" B ˆ§"I
   jAj!  k! §!A !@@  k I
@  j  °7E
  Aj" K
    j6   A:    A :    A : @ ( "E
  (" M
   k"Aj!  jAj!A ! Aÿq!@@  F
@  j-   F
  Aj" K
    j6   A:    A :    A : ^@ ( "E
  Aj! (! Aÿq!@ E
  Aj"j-   G
    6   A:   A :    A : â
~A !@@@  ( "E
  ("E
  ) "B€€€€T
  (" B ˆ§"I
  Aj!	 §!
A ! !@  k!A !@@  k I
@ 	 j 
 °7E
  Aj" M
   j"I
 	 j!	 Aj!  k" O
@ 
 A @ ( k l j"
 @ ( AJ
  A 6   A 6  ^  [!
 B€€€€T
 ) "B ˆ§! §! 
Aj!  ( "Aj!	 
(! (!A !@  I
  k!A !@  k I
@ 	 j 
 °7E
  Aj" M
  I
  I
@ E
   	 Ö6  k" I
  j!@ B€€€€T
    Ö6   j"I
 	 j!	  k!  k!  j! Aj" G
   I
@ E
   	 Ö6  G
  ( !   
6  E
  ^  t  Å6 ¹# Ak"$ @@@  ( "E
 @ ( "AJ
   (M
 
@ AJ
  A 6  A 6  ^ E
 [!  ( !   6  E
  ^  ( !  6  6  )7   ]  (  6 Aj$ ±# A0k"$ @ E
  E
 @  ( "
   6,  6(  )(7  \!  ( !   6  E
 ^ (!@ ( AJ
   j (K
   6   6$  ) 7   Aj`  ( "   ( j6 Av"   K j["  ( _  ( (!  6  6  )7   Aj`   ( ( j6  ( !   6  E
  ^ A0j$ 1@  ( "E
 @ ( AJ
  A 6  A 6  e‹@ ( "
 @ 
   B 7  b! ( !  6 @ E
  e ( ! A 6 ( A 6   ( "(6   Aj6 @ ( AJ
   ("K
    6   Aj6 @  ("  K"
   B 7  b" ( f  ( (6 ( !  6 @ E
  e ( !   (6   Aj6 ¡@@  ( "E
 @ ("   I"
 @ ( AJ
  A 6  A 6  e  6   ( "(AjO
  AtjAjA 6   ( "( kA I
   ( Aj6    z e ü# Ak"$ @@@@  ( "E
 @ ( "AJ
   (M
 
@ AJ
  A 6  A 6  e E
 b!@@  ( "
 A ! (!  Aj6     I"6  )7   d  6  (AjO
  AtjAjA 6   ( !   6  E
  e Aj$  Ù@@  ( "
 A ! (!@@  K
    Aj"z Aj"  ( "("Aj"K
  I
  k"  kK
  k I
@ At"E
  Aj" Atj  Atj ×6  ( "(Aj!  O
  AtjAj 6   (  6  §A !@@  ( "E
  ("AÿÿÿÿqE
  Aj! AjAÿÿÿÿq"Aj"Aq!A !@@ AO
 A ! Aüÿÿÿq!A !A !@  (  Fj Aj(  Fj Aj(  Fj Aj(  Fj! Aj! Aj" G
 @ E
 @  (  Fj! Aj! Aj" G
 @ 
 A    (z@@  ( "("
 A ! ! Aj"!@@ ( " F
  E
  6  Aj! Aj! Aj! Aj"
   ( "(!   k6  ( "(" (AjO
  AtjAjA 6   Ä@  ( "
 A  (!@@ E
   K
    j"I
    z  ( "(Aj" I
  I
  kAj"  kK
  k I
@ At"E
  Aj" Atj  Atj ×6  ( !   k6  ( (!  À~@ ( "E
  (" M
  ) "B ˆ§Aÿÿÿÿq"Aj  k"O
   AtjAj! At!  k! §!	A !@@   kK
@  Atj 	 °7E
  Aj" K
    j6   A:    A :    A : œ# Ak" 6@ ( "E
  (" M
   k"Aj!  AtjAj!A !@@  F
@  Atj(   (F
  Aj" K
    j6   A:    A :    A : Œ~A !@@  ( "E
  ("E
  ) "B ˆ§"Aÿÿÿÿq"Aj"	 ("
O
  At! Aj! §!
A ! 
!@  k!A !@@   kK
@  Atj 
 °7E
  Aj" M
   j"I
 Aj!  Atj! 	  k"I
@ 
 A @ ( k l 
j"
 @ ( AJ
  A 6   A 6  e  At! b"Aj!  ( "Aj! ) "B ˆ§"At! Aÿÿÿÿq! (!
 §! (!A !@@@ 	 O
   k!A !@   kK
  At"j 
 °7E
 Aj" M
 t   I
 
 I
@ E
    Ö6 
 k" I
  j!@ E
    Ö6   j"I
  k!  k!
  Atj!  Atj! Aj" G
  
 I
@ At"E
    Ö6 
 G
  ( !   6  E
  e  –@@  ( "E
  ("E
  Aj! !@@   Aj"Atj( G
 ! 
 A !  O
    z  (  6  ( "(" (AjO
  AtjAjA 6  ±# A0k"$ @ E
  E
 @  ( "
   6,  6(  )(7  c!  ( !   6  E
 e (!@ ( AJ
   j (K
   6   6$  ) 7   Ajg  ( "   ( j6 Av"   K jb"  ( f  ( (!  6  6  )7   Ajg   ( ( j6  ( !   6  E
  e A0j$    B 7    Aj6   š@  ( "  Aj"F
 @ (" ( (  @@ ("E
 @ "( "
  @  ("( G! ! 
  !  G
     (…   6   B 7  A …  % @ E
    ( …   (… AâCµ@@@  ("
   Aj"!@@  "("O
  ! ( "
  O
 ("
  Aj!AÝC" 6 B 7   6  6 @  ( ( "E
    6  ( !  ( È    (Aj6Ç@  ("E
   Aj"! !@   ( I"!  Atj( "
   F
   (I
 @@ ("
  !@  ("( G! ! 
  @ "( "
 @  (  G
    6     (Aj6  ï AâCd  B 7  A 6  Aˆ€6     Aj6AÝC"B 7  A€€€ü6 AjB 7 AÝC" 6   6 A6   ?  Aˆ€6   AjŠ  (!  A 6@ E
  ë  Aj  (‹  ¯@  ( "E
  (! A 6@ E
 @ ("E
 @ (! A 6 ( !@ E
  ^ AâC ! 
  ( ! A 6 @ E
   (AtâC AâC  ( !  A 6  E
  ëi@@ E
    ( ‹   (‹ (!  A 6@  E
   ("E
   Aj"6 
     ( (   AâC D  Aˆ€6   AjŠ  (!  A 6@ E
  ë  Aj  (‹  AâC†A !@  ("E
   Aj"! @    ( I"!   Atj( "
    F
    (I
   ("E
  (AF
   (Aj" 6 !  
   †A !@  ("E
   Aj"! @    ( I"!   Atj( "
    F
    (I
   ("E
  (AF
   (Aj" 6 !  
   ' @   "E
   (Aj" 6  
   A !@@@@ AjAI
 @@@@  ("
   Aj"!@@  "("O
  ! ( "
  O
 ("
  Aj!AÝC"A 6  6  6 B 7   6  !@  (( "E
    6 ( !  ( È    (Aj6     ( ( "
 ("
 !@  ("( G! ! 
   ("E
A   (AF  6    ("   K6 (!  6 E
  ("E
  Aj"6 
   ( (   @ "( "
 @  ( G
    6    (Aj6  ( ï (! A 6@ E
  ("E
  Aj"6 
   ( (   AâCA   A ¶@ (
     (Aj"6  6  (!@@@  ("
   Aj"!@@  "("O
  ! ( "
@  I
  ! ("
  Aj!AÝC"A 6  6  6 B 7   6  !@  (( "E
    6 ( !  ( È    (Aj6 (!  6@ E
  ("E
  Aj"6 
   ( (    ( ”@@@ AF
  E
 @@@  ("
   Aj"!@@  "("O
  ! ( "
@  I
  ! ("
  Aj!AÝC"A 6  6  6 B 7   6  !@  (( "E
    6 ( !  ( È    (Aj6@ ("E
  (AF
  ( (M
  6 (!  6@ E
  ("E
  Aj"6 
   ( (      ("   K6A 
 A  ("E
   Aj"6@ 
   ( (  A  @@  ("E
   Aj"! !@   ( I"!  Atj( "
   F
   (I
  ("E
  (AF
 @@ ("
  !@  ("( G! ! 
  @ "( "
 @  ( G
    6    (Aj6  ï (! A 6@ E
  ("E
  Aj"6 
   ( (   AâC L @    [
 A @  C   Ï]
 @  C   O`E
 Aÿÿÿÿ  Þ7" ‹C   O]E
   ¨A€€€€xX @    a
 A @  D      àÁc
 @  D  ÀÿÿÿßAfE
 Aÿÿÿÿ  Ý7" ™D      àAcE
   ªA€€€€x@  
 A A !@    -  "A-F" A+Frj"-  " E
 A !@  À" A€q
  APj"A	K
@ A/  kA
nM
 A  A
lj! Aj"-  " 
 A  k  µ~~@  
 B B !@    -  "A-F" A+Frj"-  " E
 B !@  À" A€q
  APj" A	K
@   ­"Bþÿÿÿÿÿÿÿÿ …B
€W
 B€€€€€€€€€Bÿÿÿÿÿÿÿÿÿ  A-F B
~ |! Aj"-  " 
 B  }  ³~~@ AojApK
  A :   @  B R
  A0;   A !@  BU
  A-:  B   }! A! ­!A!  !@ "Aj!  €"B U
 @ Aj"A H
   j!@@ Aq
  !  !  j     €" ~}§AêÇj-  :   A~j! E
 @  j   €"  ~}§AêÇj-  :    Aj"j     €" ~}§AêÇj-  :   A~j! 
   j jA :   :@  E
   -  "E
   !@  Àý7:   Aj"-  "
   9@  E
   ( "E
   !@  Ò$6  Aj"( "
   9@  E
   ( "E
   !@  Ó$6  Aj"( "
   ?@@  ,  þ7! ,  þ7! E
 Aj!  Aj!   F
   k?@@  ( Ó$! ( Ó$! E
 Aj!  Aj!   F
   k­@ AojApK
  A :   @  
  A0;   @@  AL
 A ! A-:  A   k! A!A!  !@ "Aj!  n"A J
 @ Aj"A H
   j!@@ Aq
  !  !  j     n" lkAêÇj-  :   A~j! E
 @  j   n"  lkAêÇj-  :    Aj"j     n" lkAêÇj-  :   A~j! 
   j jA :    A   6Àƒl	 A (Àƒl     ”  ”’‘AÝC" A6  A¼œ6   "@  ("A H
  Ï6  A6   @  ("A H
  Ï6  A6 @  ("A H
  Ï6  AâC3A !@  (AJ
    ( A€€A ¶7"6 AJ! M~# Aà k"$ @@  (" A N
 B !   A Aà Ø6"ƒ7 )! Aà j$   @  (" A N
 B  B A®7 @  (" A N
 B   A ®7*A !@  (" A H
    (  (Ù7! *A !@  (" A H
    (  (§8! ˜# Aà k"$ A !@  ("A H
 A !  A Aà Ø6"ƒ7  )Y
 A !  ("A H
 A !  A ®7BQ
 A !  (" A H
    ) "§ B ˆ§Ù7! Aà j$  #A !@  (" A H
   …7AJ! #A !@  (" A H
    ‰7E! @~# Ak"$   ) "7  ( ( !  7      !  Aj$   P# A k"$  A6  :   Aj6  ( ( !  )7   Aj  !  A j$   ~# A0k"$  A jA 6  B 7 B 7  AjA
Ÿ@ Ajì7"AI
    6  Aj6  ( ( !  )7      !  A0j$   €# A0k"$  A jA 6  B 7 B 7  AjA
™@ Ajì7"AI
    6  Aj6  ( ( !  )7      !  A0j$   «# Ak"$ £!@@  
 A !  ì7!  6   6 ( (!   )7 @@     E
 AÝC" A6   6  A°6  A€6   AÌ6  ( (  A !  Aj$    A  B    ("   ( ( )  (" ( (   ("   ( ( Y   ("   ( ( L~# Ak"$   (!   ) "7  ( ($!  7       !  Aj$   A G)  (!  A 6@ E
   ( (    .  (!  A 6@ E
   ( (    AâC~~# Ak"$   (!@@   ( (  ( (4 BR
 A !  (!  ) "7 ( ( !   7      A G! Aj$     ("   ( (( ~~# Ak"$  ) !  (!@@   ( (  ( (4 BR
 A !  (!  7 ( ( !   7      A G! Aj$  ,  (!  A 6@ E
   ( (    A|j1  (!  A 6@ E
   ( (    A|jAâC   ("   ( (( 7    ( Atj( j"(!  A 6@  E
     ( (   <    ( Atj( j"(!  A 6@  E
     ( (   AâC)   A 6  B 7   6   6  AÜ6      A 6  Ô   A 6  ÔAâC A	>A !@  ("E
    (" E
   ñ
     ( ( ! A}C    !@  ("E
    (" E
   ñ
     ( ( ! >A !@  ("E
    (" E
   ñ
     ( ( ! >A !@  ("E
    (" E
   ñ
     ( (X !    	   A ÙŽ~ Aj"! !@@ ("E
 @@   "("O
  ! ( "
   O
 ("
  Aj!AÝC" 6 B 7    6  6 @ ( ( "E
   6  ( ! ( È  (Aj6@ 
 AÝC!  )! A 6 B7 AÜ6   7 @  ×"
 A @@ ( "E
  !@   ( I" !   Atj( "
   F
 A !  (O
 A  ( (L !@ ("E
   Aj"6@ 
   ( (        6   6@  ("
 A    (v# A k"$  A6 Aø£6  )7A !@  Aj°E
    (²E
  A6 A¶£6  )7   °! A j$  7  (!AÝC" A 6  B7   6   6  AÜ6      ,@  ("
   ­B€€€€€€€€€„ ­B †  5„/@    ( (T " E
     (Aj"6 
    /@    ( (T " E
     (Aj"6 
       J# Ak"$  B 7  Aj6    Aj  ( (L !  Aj (ì Aj$   J# Ak"$  B 7  Aj6  A Aj  ( (L !  Aj (ì Aj$        ( (  A  A  C     A /@    ( (X " E
     (Aj"6 
    /@    ( (X " E
     (Aj"6 
     A    A      ( ($  A      ( ((  A      ( (,  A      ( (0  A      ( (4  A      ( (8  A      ( (<  A      ( (@  A      ( (D 3 @  (" E
 AÝC   Å"   (Aj"6 E
        B 7 @ (E
    ) 7       6   6   o# AÐ k"$    6 A jA AŠè Ajâ7@@ A jì7" 
 A !    6L  A j6H  )H7 Aj\!  AÐ j$   ®# A k"$   6  6A !@A A    8"AH
 A ! A 6 Aj Aj j (A  Aj"Ø6  ("6 (    8@ ("E
  Ajì7! Aj k (! A j$  &# Ak"$   6   ú! Aj$  s# Ak"$ @@ 
   A 6  ì7!  A 6  E
   6  6  )7  \!  ( !   6  E
  ^ Aj$   \~# Ak"$   A 6 @ (E
   ) "7   7 \!  ( !   6  E
  ^ Aj$   ­~# A k"$   A 6 @ (" ("j" I
 @ E
  [!  ( !   6 @ E
  ^  ( !  ) "7  7  Aj]  ( !  ) "7   7   ` A j$    î	~~# Ak"$ A !  A 6 @@ ("AÿÿÿÿqE
  ( !@@ Aÿÿÿÿj"Aÿÿÿÿq"
 A! AjAþÿÿÿq!A !A!A !@B !	B !
@ AqE
  (" j"A   O"­B † ­„!
@ 
§AqE
  Aj( " 
B ˆ§j"A   O"­B † ­„!	 Aj! 	B ˆ§! 	§! Aj" G
 @ Aq
 B !	@ AqE
  (" j"A   O"­B † ­„!	 	B ˆ§! 	§! AqE
 	B€€€€T
  [!  ( !   6 @ E
  ^ ("AÿÿÿÿqE
  ( " Atj!A !@  ( !  ) "	7   	7   ` ( j! Aj" G
  Aj$    €# A k"$   A 6 @@@ (4"AqE
 @ (0" ("O
   60 ! Aj!@ Aq
 A ! A :  Aj! Aj! (!@@  ( "k"AøÿÿÿO
 @ AI
  ArAj"AZ!  A€€€€xr6  6  6  :  Aj! 
A !Å6    ×6  jA :  @ ( , " A H""E
   6  ( Aj 6  )7  \!  ( !   6  E
  ^@ , AJ
  (L A j$   ( @@ E
  -  
  i      ì7u  $@ ("
   i     (  u  F@  ( " ( "F
 @ E
   ( Aj6   ( !   6  E
  ^  6@  (  ( "F
  A 6   ( !   6  E
  ^   @ E
     ì7v  (# Ak"$   :    AjAv Aj$    @ ( "E
    Aj (v  @ ("E
    (  v  ? @@  ( " E
   (
@ 
 A -  E@ 
 A   Aj é7E’A !@   ( " rE
   AjA²œ  " F
 A !A !@ E
  ì7!@  E
   (!@@    I" 
 A !A!    °7"A H
 E  Iq! o (!@@  ( " 
 AA  !@   ("  I"E
   Aj (  °7"
A !  F
 AA  I! Avj@  ( " 
 AA  (@@ ("  ("  I"E
   Aj (  °7"
A !  F
 AA  I! @  ( "  ( "G
 A A !A !@  E
   (!@ E
  (!@@    I"
 A !   AjA²œ   AjA²œ  °7" A N
 A  E  IqŸ@  ( " 
  (E@  (" (F
 A @ 
 A  Aj!  ( ! !@@ E
@  -  " -  "F
  ý7 ý7F
 A  Aj! Aj!  Aj!  Aj"
 A é# A k"$ @@  ( " 
 A ! @ 
     ( Aj6    ("6   Aj6 Aj Aj÷@@ ("E
   (" O
   k"Aj  O
  Aj  I
A !  Aj  j ø@ (" 
 A !    6  (6  )7  \!  A j$   Í# A k"$ @@  ( " 
 A ! @   ("G
     ( Aj6   6   Aj6 Aj Aj÷@@ (" E
  Aj (I
A !  Aj   ø@ (" 
 A !    6  (6  )7  \!  A j$   ì# A k"$ @@  ( " 
 A ! @  (" G
     ( Aj6   6   Aj6 Aj Aj÷@@ ("E
   k" (" O
  Aj  O
  Aj  I
A !  Aj  j ø@ (" 
 A !    6  (6  )7  \!  A j$   -@  ( "E
  ("E
    l  ( Ajš) @ ( "
   A²œA  9   Aj ( 9ó@  ("
 A   ( !  Aq!A !@@ AO
 A ! Axq!A !A !@ Al  -  jAl  Aj-  jAl  Aj-  jAl  Aj-  jAl  Aj-  jAl  Aj-  jAl  Aj-  jAl  Aj-  j!  Aj!  Aj" G
 @ E
 @ Al  -  j!  Aj!  Aj" G
  6   A 6  B 7  B 7  AÀž6   AjB 7   A jA 6   Q   A 6  B 7  A 6  B 7  AÀž6    ( "6@ E
   ( Aj6   A 6   ÷  A6@  ("  ("F
 @@ ( (AG
  A 6  Aj" G
   (!  A 6@ E
  ë@@  ("E
  !@   ("F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
   (!   6   ( kâC  Ô    —A$âC A   	   A Ù 
# Ak"$  Aj"! !@@ ("E
 @@   "("O
  ! ( "
   O
 ("
  Aj!AÝC" 6 B 7    6  6 @ ( ( "E
   6  ( ! ( È  (Aj6A$ÝC"A 6 B7 B 7 AÀž6  AjB 7  A jA 6 @@  ("	  ("
F
  Aj! AjAj!@@@ ( "E
  	( ! !@   ( I" !   Atj( "
   F
   (O
  6 B 7 Aj (  ê  	( "  Aj ( (L "6 @ E
 @@ (" (O
  A 6  ( !  A 6  ( !   6 @ E
  (" E
   Aj" 6  
   ( (   Aj!  !  6 ( ! A 6  E
  ("E
  Aj"6 
   ( (   Aj (ì 	Aj"	 
G
  Aj$   ‡@@@  (  ( "kAu"Aj"A€€€€O
 A !@  ( k"Au"   KAÿÿÿÿ AüÿÿÿI"E
  A€€€€O
 AtÝC! ( ! A 6   Atj" 6   Atj! Aj!@  ("  ( "F
 @ A|j"( ! A 6  A|j" 6   G
   (!  ( !   6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
 @ E
    kâC Å6 ð  }  B 7   AjB 7 @ (" ("kAG
 @@@@  F
  ( " ( ( ! (! (!   8   kAK
   A 6 Aj( " ( ( ! (! (!   8  kAK
  A 6C    ! Aj( " ( ( ! (! (!   8C    !  kA
I
  Aj( " ( ( !   8?}C    !@   (  (" kAuO
    Atj( "   ( ( ! È}}}}}}@@@@ (" ("kAF
 C    !C  €?!C    !C  €?!C    !@  G
 C    ! ( " ( ( ! ( ("kAO
C    !C    !C    !C    !	C    ! Aj( " ( ( !@@@ ( ("kA	O
 C    ! Aj( " ( ( ! ( ("kA
O
C    !C    !	 Aj( " ( ( !@ ( ("kAO
 C    !	 Aj( " ( ( !	 ( ("kAI
  Aj( " ( ( !   8   	8   8   8   8   8 @@@ ( ("F
 A !@@@  Atj( "
  E
  (Aj"6 E
 Ö! ("E
  Aj"6@ 
   ( (  @ E
  ("E
  Aj"6 
   ( (    G
  F
 Aj" ( ("kAuI
   A :    A :     6   A: yA !@@   (  (" kAuO
    Atj( " E
     (Aj"6 E
  Ö!  ("E
   Aj"6 
     ( (    )# Ak"$  Aj   ¡ - ! Aj$  JA !@   (  (" kAuO
    Atj( " E
     (Aj"6  ! 
   JA !@   (  (" kAuO
    Atj( " E
     (Aj"6  ! 
   yA !@@   (  (" kAuO
    Atj( " E
     (Aj"6 E
  Ö!  ("E
   Aj"6 
     ( (    <A !@   (  (" kAuO
    Atj( "   ( ( ! <A !@   (  (" kAuO
    Atj( "   ( ( ! F @   (  (" kAuO
    Atj( " E
   çE
     ( ( A G! <A !@   (  (" kAuO
    Atj( "   ( ( ! øA !@@   (  (" kAuO
    Atj( " E
     (Aj"6 E
  Ö!  ("E
   Aj"6@ 
     ( (   E
 @@  ( (, "E
   (Aj" 6  
@  ( (@ " 
 A !  á! (" E
   Aj" 6  
   ( (    	    «ÓA !@@   (  (" kAuO
    Atj( " E
     (Aj"6 E
  Ö!  ("E
   Aj"6@ 
     ( (   E
 @  ( (@ "E
   (Aj" 6  E
 (" E
   Aj" 6  
   ( (    	    ­ÓA !@@   (  (" kAuO
    Atj( " E
     (Aj"6 E
  Ö!  ("E
   Aj"6@ 
     ( (   E
 @  ( ($ "E
   (Aj" 6  E
 (" E
   Aj" 6  
   ( (    	    ¯“A !@@   (  (" kAuO
    Atj( " E
     (Aj"6 E
@  ï"E
   (Aj"6 E
  ("E
   Aj"6 
     ( (    “A !@@   (  (" kAuO
    Atj( " E
     (Aj"6 E
@  õ"E
   (Aj"6 E
  ("E
   Aj"6 
     ( (    }@  ( 
 @  ("  ("F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
    6 Œ@  ( 
 @   ("  ("kAuO
 @  Atj"Aj" F
 @ ( ! A 6  ( !  6 @ E
  ("E
  Aj"6 
   ( (   Aj! Aj" G
   (!@  F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
    6 Ó@  ( 
 @   (  ("kAuO
   Atj( "E
  ñ
 @  ( At"j( "E
   (Aj"6 E
  ’  ( j( "  ( (P !  ( j"( !   6   E
   ("E
   Aj"6 
     ( (   ¢ @  ( 
  E
  (
  ó
 @@@   (  (" kAuO
    Atj"( !   6   E
  ("E
   Aj"6 E
 (" E
   Aj"6 !  
    ( (   ¨ @  ( 
  E
  (
  ó
 @@@   (  (" kAuO
    Atj"( !   6   E
  ("E
   Aj"6 E
 (" E
   Aj"6 ! A ! 
    ( (         ¹È# Ak"$   6@  ( 
  E
  (
  ó
 @@@   (  ("kAuM
 A ! A 6 !   Aj  Atj Ajº (!  A 6 !  E
  ("E
   Aj"6@ 
     ( (   ! Aj$   ­# A k"$ @@@@@  ("  ("O
 @  G
  A 6  ( ! A 6  ( !  6 @ E
  ("E
  Aj"6 
   ( (     Aj6 !@ A|j" O
  ! !@ A 6  ( ! A 6  ( !	  6 @ 	E
  	("E
 	 Aj"6 
  	 	( (   Aj! Aj" I
    6@  AjF
 @ A|j"( ! A 6  A|j"( !  6 @ E
  ("E
  Aj"6 
   ( (    G
  ( ! A 6  ( !  6  E
 ("E
  Aj"6 
  ( (     ( "kAuAj"A€€€€O
   Aj6@@  k"Au"   KAÿÿÿÿ AüÿÿÿI"
 A ! A€€€€O
 AtÝC!  6    kAuAtj"6   Atj6  6 Aj ½   Aj ¾!@ (" ("F
 @  A|j"6 ( ! A 6 @ E
  ("E
  Aj"6 
   ( (    ("G
  ("E
   ( kâC A j$   Å6 ð 
    ¼¸# Ak"$   6@  ( 
  E
  (
  ó
 @@  ("  (O
   6    Aj6  Aj Aj! (!   6 A 6 E
  (" E
   Aj" 6  
   ( (   Aj$   ü	@@  ("  (G
 @  ("  ( "M
    kAuAjA~mAt"j!@  F
 @ ( ! A 6  ( !  6 @ E
  ("E
  Aj"6 
   ( (   Aj! Aj" G
   (!   6    j6@@@A  kAu  F"A€€€€O
  At"ÝC" j!	  A|qj!
  F
 
  kj! 
!@ ( ! A 6   6  Aj! Aj" G
   (!   	6  (!   6  (!   
6  ( !   6   F
@ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
  ð    	6   
6   
6   6  E
    kâC  ("A 6  ( ! A 6  ( !  6 @ E
  ("E
  Aj"6 
   ( (      (Aj6 ý ("!@@   ( "F
  ! !@ A|j"A 6  A|j"( ! A 6  ( !  6 @ E
  ("E
  Aj"6 
   ( (    G
   6 (!@   ("F
 @ A 6  ( ! A 6  ( !  6 @ E
  ("E
  Aj"6 
   ( (   Aj! Aj" G
  (!  6  ( !   6   6  (!   (6  6  (!   (6  6  (6   Ÿ# A k"$  A6 Aó6  )7@@@  Aj°
 A !@  (  ("F
 A !@@  Atj( "E
   (Aj"6 E
    ( (H ! ("E
  Aj"6@ 
   ( (  @ E
  Aj"  (  ("kAuO
A ! A6 Aíò6  )7   °! A j$   8   6   (Aj"6@ E
   ( " ( Aj6        6   ( Aj6   \  ( " ( Aj6   ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     E   A 6  B 7    ( 6    (6   (6 A 6 B 7    6  @  ( "E
    6 J  >~@@  ­ ­~"B ˆ§
  §­ ¬~"BùÿÿÿT
  B †B€€€ð |B#ˆ§<~@@  A H
  ¬  ­~"BáÿÿÿT
  B †B€€€ð|B#ˆ§AüÿÿÿqW~@@ ­ ­~"B ˆ§
  §­ ¬~"BùÿÿÿT
  A :    A :    B †B€€€ð |B#ˆ>   A: V~@@ A H
  ¬ ­~"BáÿÿÿT
  A :    A :    B †B€€€ð|B#ˆ§Aüÿÿÿq6   A: D   B 7  A¤Ÿ6   A 6(  Bÿÿÿÿ7   AjB 7   AjB 7   AjA 6   P   A 6(  Bÿÿÿÿ7    6   6   6   6   6   6   6  A¤Ÿ6        î~# Ak"$ @@ ( " AjG
    ($6    ((6@@ A H
   L
@  ( ( 
   B 7 A ! A 6 @  N
 @ Aj  ( (   ( Aj"6   H
  Aj  ( (   )"7$  ( Aj6    B ˆ>   >  Aj$ Ù# Ak"$ A !@  ( " F
   AjF
 A !@@ A H
   L
    ( ( E
A !  A 6   B 7$A !  N
 @ Aj    ( (    )7$    ( Aj"6 @ E
 @  ( ( E
 A!  ( !  H
  Aj$  ¾~A !	@@   "AH
    "AH
   rAÿÿK
 AØ ÝC!	  ) !
 	    AAA ÆÊ"	 
78 	 : 6 	 : 5 	 : 4 	A 60 	 6, 	Að¦6  	A 6H 	B 7@@ 	("
  	A 6T 	B 7L 	 AL
 	 AV"6@ 	  j"6H A  Ø6 	 6D 	A 6T 	B 7L 	("E
  AL
 	 AV"6L 	  j"6T A  Ø6 	 6P 	Å6 ²# A k"$   (!  ( !@@@@@ 
 A ! A !	 AL
 AV"	Aÿ Ø6 j!   6 AH
   	k!
 At!A !@   lj! @@ E
   Aÿ Ø6!   
6  	6  )7   Aj   Aj Ñ 	   Ö6  
6  	6  )7    Aj    Ñ Aj" F
  Å6  (!@ 	E
  	J A j$  Ï~# A0k"$ @ ( " N
  ) "B ˆ§!	 §!
A!A!@A!@@@@@@ "
A H
  
Av" 	O
 
 j-   
AsAqvAq!  	6,  
6(  )(7 ! !@ Aj  
Aj As"Ò" N
 @@ Aq G
  !  	6$  
6   ) 7 Aj  Aj Ò! ! !  N
   	6  
6  )7    Aj Ò! !  Aj"6 A !   Am"j-   At kAjvAq
  N
  Aj"6   N
   Am"j-  !  Aj"6 A Am"At kAjt   j-  q!@  At kAjvAqE
 AA !@@@@@ E
 AÀŸA¢ Aq"!AÅAÆ !A !@A!A !A !@ !  j-  "AÿF
  N
   Am"j-  !  Aj"6   At kAjvAq Atr! !@@  Al j"N
 @  O
   j-  F
 Aj" H
  Aj!  O
 Aj" O
 Aj" O
  j-  At  j-  r" j! A?K
    N
  Aj"6 @   Am"j-   At kAjvAqE
  Aq
A     H A H" 
A  
A J"L
 Aq!@ Av" Aj"Av"G
  Aq" I
  k!
  j"-  !A !@  kAq"E
 @ AA ktj! Aj! Aj" G
 @ 
AI
 @ AA ktjAA ktjAA ktjAA ktj! Aj! Aj!  G
   :    j"
-  A Astj! AF
 AA ktj! AF
 AA ktj! AF
 AA ktj! AF
 AA ktj! AF
 AA ktj! AF
 AA ktj" Aj AF!  N
  Aj"6   N
   Am"j-  !  Aj"6 A Am"At kAjt   j-  q!@  At kAjvAqE
 AA~ !@ E
   N
  Aj6 AA}   Am"j-   At kAjvAq!  N
  Aj6 @   Am"j-   At kAjvAqE
   A
j6  
!	  Aj6 
 Aj! 
 :  @ Ao"A H
   j"-  !@@ 
  A€s!@ AG
  AÀ j!@ AG
  A j!@ AG
  Aj!@ AG
  Aj!@ AG
  Aj! AA AFj!  :    AjM
   Asj"E
  
AjA  Ø6  H
  
Avj"A H
  
j!@@ E
 AÆ!A¢!AÅ!AÀŸ!A     H A H" 
A  
A J"L
  Aq!@ Av" Aj"
Av"G
  
Aq"
 I
 
 k!  j"-  !A !@  kAq"E
 @ AA ktj! Aj! Aj" G
 @ AI
 @ AA ktjAA ktjAA ktjAA ktj! Aj! Aj!  
G
   :    j"-  A Astj!@ AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj" Aj AF!  :  @ 
Ao"A H
   j"-  !
@@ 
  
A€s!@ AG
  
AÀ j!@ AG
  
A j!@ AG
  
Aj!@ AG
  
Aj!@ AG
  
Aj! 
AA AFj!  :    AjM
   Asj"E
  AjA  Ø6A !@A!A !A !@  j-  "AÿF
 ( " N
   Am"
j-  !  Aj6   
At kAjvAq Atr! !@@  Al j"N
 @  O
   j-  F
 Aj" H
  Aj!  O
 Aj" O
 Aj" O
  j-  At  j-  r" j! A?K
    Aj! A H
  j!@ E
 A     H A H"
 A  A J"L
  Aq!@ Av" 
Aj"Av"G
  Aq" I
  k!  j"-  !A !@ 
 kAq"E
 @ AA ktj! Aj! Aj" G
 @ AI
 @ AA ktjAA ktjAA ktjAA ktj! Aj! Aj!  G
   :    j"
-  A Astj!@ AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj" Aj AF! 
 :  @ Ao"A H
   j"-  !@@ 
  A€s!@ AG
  AÀ j!@ AG
  A j!@ AG
  Aj!@ AG
  Aj!@ AG
  Aj! AA AFj!  :    AjM
   Asj"E
  
AjA  Ø6  N
  j!@ Aq
 A     H A H" 
A  
A J"L
  Aq!@ Av" Aj"Av"G
  Aq" I
  k!  j"-  !A !@  kAq"E
 @ AA ktj! Aj! Aj" G
 @ AI
 @ AA ktjAA ktjAA ktjAA ktj! Aj! Aj!  G
   :    j"-  A Astj!@ AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj! AF
  AA ktj" Aj AF!  :  @ Ao"A H
   j"-  !@@ 
  A€s!@ AG
  AÀ j!@ AG
  A j!@ AG
  Aj!@ AG
  Aj!@ AG
  Aj! AA AFj!  :    AjM
   Asj"E
  AjA  Ø6  N
 
 N
 As! ( " H
  A0j$ Ò@@@  N
  Aj!@  Am"Atk"E
    (O
  (  j-   sAÿ vq"
 Aj! Am! Aj"Am!@ AÀ H
   Axj"N
 AÖ¤AÞ¤ !  ( !  (!@  K
  j)   )  R
 Aj" H
   N
    ("  K!  ( !  Aÿq!@  F
@   j-  " F
   At  sAÿqAæ¤j-  j"  H Aj" G
    At Aæ¤j-  j@  B 7$@  (L"E
    6P J@  (@"E
    6D J  ËF  B 7$@  (L"E
    6P J@  (@"E
    6D J  ËAØ âC  (<"  (0AjAm"    I1@  (P"  (L"F
  Aÿ  kØ6  A 60Aé
# A k"$ @ (<At"AL
  (0"   J! A0j! (8! !@@@@  F
  Aj"	6  Am"
At k! 	!  
j-   AjvAqE
 @ 	 kAJ
   6  !	 	 H
  B 7 @ (D"	 (@"F
  Aÿ 	 kØ6@@ (,"	AJ
  (@! (8!	 (P!
  (L"6  
 k6 (!
  )7  	     
Ñ AÌ j (@" (D"	 	 kØ (8!@ 	
     (@ (Ù  (0"	Aj60 (@!
@@  	Am"j-   At 	kAjvAqE
     
 (Ù (P!	  (L"6  	 k6 (!	  )7    
 Aj 	Ñ AÌ j (@" (D"	 	 kØ@ - 5AG
  (0"   J! (8! !@  F
  Aj"	6  Am"
At k! 	!  
j-   AjvAqE
  	 kAJ
   6 @ - 4AG
  ( " N
 @@  AjAxq"
N
  (8! (<!@ Am"	 O
  	j-   	At kAjvAq
 Aj" 
H
   
6  A : 4 (@!@ - 6AG
  Aq
 (D k"AI
  !@ A|j"AvAjAq"
E
 A !	 !@  ( As6  Aj! 	Aj"	 
G
 @ AI
   A|qj!
@  ( As6  Aj"	 	( As6  Aj"	 	( As6  Aj"	 	( As6  Aj"	 	( As6  Aj"	 	( As6  Aj"	 	( As6  Aj"	 	( As6  A j" 
G
  (@! (D!   6     k6 A j$   @   ("  ( "kK
 @   (" k"M
   j!@  F
    ×6  (! !@  F
  !@  -  :   Aj! Aj" G
      kj6  k!@  F
    ×6    j6@ E
    6 JA !  A 6  B 7 @ AL
    At"   KAÿÿÿÿ AÿÿÿÿI"AV"6   6     j6@  F
 @@  kAq"
  !A ! !@  -  :   Aj! Aj! Aj" G
   kAxK
 @  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
    6Å6 ‘
A !A!@@ ! ( " N
AÀŸA¢ Aq"	!
AÅAÆ 	!A !@@@@@@A!
A !A !@ ! 
 j-  "AÿF
  N
   Am"j-  !  Aj"6   At kAjvAq Atr! 
!@@ 
 Al 
j"N
 @  O
  
 j-  F
 Aj" H
  Aj!
  O
 Aj" O
 Aj" O
 
 j-  At 
 j-  r" j! A?K
   j! 	
A     H A H" A  A J"J
    J!@  F
  Aj"6  Am"At k!
 !   j-   
AjvAqE
   Aq!@ Av"
 Aj"
Av"G
  
Aq"
 I
 
 k!  
j"-  !
A !@  kAq"E
 @ 
AA ktj!
 Aj! Aj" G
 @ AI
 @ 
AA ktjAA ktjAA ktjAA ktj!
 Aj! Aj!  
G
   
:    
j"-  A Astj! AF
 AA ktj! AF
 AA ktj! AF
 AA ktj! AF
 AA ktj! AF
 AA ktj! AF
 AA ktj" Aj AF!   :  @ 
Ao"A H
   j"
-  !@@ 
  A€s!@ AG
  AÀ j!@ AG
  A j!@ AG
  Aj!@ AG
  Aj!@ AG
  Aj! AA AFj! 
 :    
AjM
   
Asj"E
  AjA  Ø6 As!  H
  ~@A AF A	J"	
 AÀ ÝC!  ) !
          ÅÊ" 
70A ! A 6, AŒ§6 @ ("E
  AW!  6<  68 Að ÝC!  ) !
          ÅÊ" 
70A ! A 6, AŒ§6 A !@ ("E
  AW!  6<  68 B 7P  	: @ A¨§6  AØ jB 7  Aà jB 7  Aè jB 7    (  l l""6L   ( "6H   ( "6D    Å"6P@ E
  AW! (X!  6X@ E
  J  6\@@ (P"
 A ! AW! (`!  6`@ E
  J  6dA !@ (PAj"E
  AW! (h!  6h@ E
  J  6l þ~# Ak"	$ @@@@@@@@ E
 A8ÝC! ) !
 B 7  
7  AjB 7  AjB 7 A AW! A 6, A	: )  : ( A 6$  6 A'AW! A'64  60 Aj! ($! ( !
@@ ( ("k"AÿK
  A€ kÜ A€F
   A€j6@@@@ (" - )"j" ("AtK
 A!A !@ Av!@@ Aq"
 A !  O
 (  j-  AA k"tAsq  k"Aÿqt! Aj!@@ AÿqAK
   O
 (  j-  A kAÿqv r!  O
 ( " j-   AxjAÿq"t r! E
  Aj" O
  j-  A kv r!  6@@@@@@ AÿK
 @ (" ( ("k"I
  Av"  kAj"  K" j" I
  O
   Ü (! (!  j :    (Aj6 AF
 (," - (jAþF
  Aj6,  (4O
 (0 Atj  Atr6 @@ (, - (j"AþG
 A
!@ Aþ
F
  AþG
A!A!  : ) !@@ A€~j  A 6, A	: )A! AF
 A 6@@ Aþ}j" (,"I
 @@ ($
 A ! A6 E
 
 :   (,! ($! ( ! !@ Aþ}j"A H
  !  O
  (4! (0!@  O
 (" O
  Atj( !  Aj6  j :   Av! A€€ˆI
 Aþ}j" (,I
  (" O
  Aj6  j :  A ! ($! ( ! !@ A H
  (4! (0!@  O
 (" O
  Atj( !  Aj6  j :   Av!@ A€€ˆI
  Aþ}j" (,I
 (!  O
   Aj6  j :   (" (j" I
@  ( ("k"M
  Av"  k"  K" j" I
  O
   Ü (!A !A ! E
@  Asj" O
 ( (j j 
 j-  :   ("! Aj" O
    6  ( j6 Aj" O
 
 j-  ! (,!@ A‚I
  Aþ}j O
  - (jAþF
  Aj6,  (4O
 (0 Atj At r6 @@ (, - (j"AþG
 A
!@ Aþ
F
  AþG
A!A!  : ) ! ! (" - )"j" ("AtM
  ("E
   (" ("k"M
   kÜ (! (!A ! 	Aè jA 6  	B 7`A!   	Aà jAÃ 	(`"E
 	 6d JA !  j   I! B 7 (!
 A 6 (AjAv!A! ) !
AA8W"AÌ 6$ AÍ 6  Až”A8Þ,  
7   
B ˆ§At "A€­â A€­âI!A !A !@ E
  AV"A  Ø6 j! 	A 6„ 	B 7| 	 6X 	 6T 	 6P   k"6  6 (! Aß,!
 (" A€€€€ A€€€€I"I
@@  A€€€€ A€€€€I k"I
@  F
   jA   kØ6 	(„! 	(€!@@ (" 
rE
   O
 A 6 B 7   	(P6   	(T6  	(X6 	 Aj"6€@@  O
  A 6 B 7   	(P6   	(T6  	(X6 	A 6X 	B 7P Aj! 	Aü j 	AÐ jß! 	 6€@@ 
 A !A ! AV"A  Ø6 j!@ 	(P"E
  	 6T J 	 6X 	 6T 	 6P   k"6  6 (! Aß,!
 (" A€€€€ A€€€€I"O
 	 	Aü j 	AÐ jß"6€ 	(P"E
  	 6T J 	(€! ("
A€€€€ 
A€€€€I! (!@@  	(|"kAG
  
 K
@@  (" ( "k"M
    kÜ 	(|"(! ( !  O
    j"6 	 6t 	 6p 	 (6x A 6 B 7  	AÐ j 	Að j Ã@@ 

 A !A ! AV"A  Ø6 j! 	(|! 	(€!@  F
   k!
A ! !@  Alj"A 6 (! ( ! B 7  
 A   Aj" 	(€ 	(|kAmIk" 
 I"  kK
@ E
    Ö6@ E
  J  j! 
 k!
  	(€ 	(|"kAmI
  	 6x 	 6t 	 6p 	AÐ j 	Að j Ã@ 	(p"E
  	 6t J@ 	(|"E
  !@  	(€"F
 @@ Atj"( "E
  Axj 6  J !  G
  	(|! 	 6€  	(„ kâC á, J 	(P! 	A 6P 	(X!
 	(T! 	B 7T 	(\! 	AÐ jÄA !A !A !
 (0! A 60@ E
  J ( ! A 6 @ E
  J@ ( "E
   6 J A8âC E
@@@@A AF A	J   	 
6L 	 6H 	 6D   	AÄ j Ã 	(D"E
 	 6H 	AÐ j   Ç@ 	- TAG
  	(P"E
  Aj"E
    k"j" n!  K
  ­ ­~"
B ˆ§
A !A !A !@  p" kA   
§j"E
  AL
 AV"A  Ø6 j!  k!  lAjAm! ! !A !
A !@ 	 "6€ 	 "6| 	 6p 	 
6Œ 	 6ˆ 	 	)|7 	 6t 	 	)p7 	 	)ˆ7  	Aj 	Aj 	 Aj"   I" à  Aj"
I
  I
  j!  k!  
k!  
j! !
 ! Aj" I
  	 64 	 60 	 6,   	A,j Ã 	(,"E
 	 60 J 
 	 
6@ 	 6< 	 68   	A8jAÃ 	(8"E
 	 6< 	AÐ j   Ç@ 	(PA  	- T"E
   k"E
  !@ 	    I"6T 	 6P 	 	)P7 	Aj   á  j!  k"
  	 
6( 	 6$ 	 6    	A j A Ã 	( "E
 	 6$Å6   E
 J 	Aj$ @  ("  ("k I
 @ E
  A  Ø6 j!   6@   ( "k" j"AL
 A !@  k"At"   KAÿÿÿÿ AÿÿÿÿI"E
  AV!  j!  j"A  Ø6 j!@  ("  ( "F
 @@  kAq"
  !A ! !@ Aj" Aj"-  :   Aj" G
 @  kA|K
 @ Aj Aj-  :   A~j A~j-  :   A}j A}j-  :   A|j" A|j"-  :    G
   ( !   6   6   6 @ E
  JÅ6   J
   AXÁ@@  (  ( "kAm"Aj"AÖªÕªO
 A !@  ( kAm"At"   KAÕªÕª AªÕªÕ I"E
  AÖªÕªO
 AlÝC!  Alj" ( 6   (6  (6 A 6 B 7   Alj! Aj!@@  ("  ( "G
  !@ Axj"B 7  Atj" Atj"( 6   Axj( 6  A|j A|j"( 6  A 6  B 7  ! !  G
   (!  ( !   6   6   (!   6@  F
 @@ Atj"( "E
  Axj 6  J !  G
 @ E
    kâC Å6 ð Ø~@@ ("E
   AjK
  ( "Aj!@@@@@ -  Aj  E
 ) "B ˆ§!  (!  ( !	 §!
A !@@@@@@  I
   k" O
 	 j-  !A !  B€€€€Z
A !A ! B€€€€Z
A ! A !  I
  O
 
 j-  ! A !  O
 
 j-  ! 
 j-  !   F
 	 j  j-       Aÿq" Aÿq"
j Aÿq"k" k" Au"s k"  k" Au"s k"K"      
k" Au"s k" K  Kj:   Aj" G
    ( I
 E
  (   ×6 E
  (!  ( !A !@A ! @  I
   k"  O
   j-  !   F
  j  j-    j:   Aj" G
   E
 ) "B ˆ§!  ( !  (!  §!A !@A !@ B€€€€T
   O
  j-  !   F
  j  j-   j:   Aj" G
   E
 ) "B ˆ§!  (!  ( ! §!
A !@A ! @  I
   k"  O
   j-  ! A !@ B€€€€T
   O
 
 j-  !  F
  j  j-   Aÿq  AÿqjAvj:   Aj" G
   ·@@@ AG
   ("At"AL
   l"  H"AH
  ( !A !A !A!@ Av"  O
  O
   j" -  "A Aq"As"	tr A~ 	wq  j-   Asv  	vsAq:    ! ! Aj" G
    lAm!  (!@ AF
   O
  ( !  !@  k" O
   j"	 	-     j-  j:   Aj" G
   Aj" O
   ( !  !@  k"	 O
 	Aj" O
  O
   j" -  At   j"-  r   	j-  At   j-  rj"	Av:    	:   Aj! Aj!  I
  ‚~# Ak"$ @@ ("Aèn" j" I
  AtI"E
   A 6  B 7 @@ At Aj"
 A !A ! AL
   AV"6     j"6 A  Ø6   6 ) !   k"6  Aj § B ˆ§¹,!@@A  ( " M
     kÜ  O
     j6 Aj$  Å6 S  B 7$  AŒ§6   (8!  A 68@ E
  J  (,!  A 6,@ E
  á, J  ËY  B 7$  AŒ§6   (8!  A 68@ E
  J  (,!  A 6,@ E
  á, J  ËAÀ âC
   (,(`AA8W"AÌ 6$ AÍ 6  Až”A8Þ,  (,!   6,@ E
  á, J  (,"
 A    )07 A› (8! (," (<"6  6 (! Aß,@ (" A€€€€ A€€€€I"I
   A€€€€ A€€€€I k"I
 @  F
   jA   kØ6   )87  ¨  B 7$  (h!  A 6h@ E
  J  (`!  A 6`@ E
  J  (X!  A 6X@ E
  J  B 7$  AŒ§6   (8!  A 68@ E
  J  (,!  A 6,@ E
  á, J  Ë
   èAð âCgAA8W"AÌ 6$ AÍ 6  Až”A8Þ,  (,!   6,@ E
  á, J  (,"
 A    )07   A 6TAÉ~~~# Aà k"$ @@@ (" (P"G
 @@ - @  (H (D (LÅ! (D! (H! (h! (," (l"	6  6 (! Aß,  lAjAm! (" A€€€€ A€€€€I"I
 	 A€€€€ A€€€€I k"I
@ 	 F
   jA  	 kØ6  )8"
7X  )h"7P )X!  
7  7  7H  7 Aj Aj Aj  à (P" (<K
 (\ I
 E
 (X (8 ×6 (8! (," (<"	6  6 (! Aß, (" A€€€€ A€€€€I"I
 	 A€€€€ A€€€€I k"I
@ 	 F
   jA  	 kØ6 (P" (<K
 (8!	  6\  	6X (! (!	 (!  )X7  A j  	 á@ (T"   I"	E
  (d"  k"I
 	  kK
 (< 	I
 (8 (` j 	×6  (T 	k6T  	k! (H (D (LÅ! (D (HlAjAm!@@ - @  E
@ (`! (,"	 (d"6 	 6 	(! 	Aß, 	("	 A€€€€ A€€€€I"I
  	A€€€€ 	A€€€€I k"	I
@  	F
   	jA   	kØ6  )`"7X (L!	 (D! (H!  7@ AÀ j   	á  (P"  I"	 (dK
 (<" ( k"I
  k 	I
@@ 
 A ! (8 j (` 	×6 (P!   	k (Tj6T  	k"
   E
 @ (h! (,"	 (l"6 	 6 	(! 	Aß, 	("	 A€€€€ A€€€€I"I
  	A€€€€ 	A€€€€I k"	I
@  	F
   	jA   	kØ6  )`"
7X  )h"7P )X!  
78  70  7H  7( A8j A0j A(j  à (\ (d"	I
@@ 	
 A ! (X (` 	×6 (d!  (P"  I"	 K
 (<" ( k"I
  k 	I
@@ 
 A ! (8 j (` 	×6 (P!   	k (Tj6T  	k"
    )87  Aà j$  …# Ak"$ A ! A 6@  ("AÿÿÿÿqE
   ( " Atj!@@ ( "AÿÿÃ K
 @ Aÿ K
  Aj À† AjAAA A€€I A€I" Aº§j-     Aj"Al" vrÀ†A  t! A !@ Aj  Aj q"  Av" nA€r†  !  Aj" G
  Aj" G
  (! Aj$  Ç (!  AjA 6   B 7 @ E
    î AÿÿÿÿqE
  ( " Atj!@@@ ( "A€€|jAÿÿ?K
  Aÿq! A€€<jA
vAÿqA€°r!@@@@  , "A H
 A! AF
   AjAÿ q:   !  ("  (AÿÿÿÿqAj"G
   A  A A ï !   Aj6  ( ! A€¸r!  Atj" ;  AjA ; @@@@  , "A H
 A! AF
   AjAÿ q:   !  ("  (AÿÿÿÿqAj"G
   A  A A ï !   Aj6  ( !  Atj" ; @@@@  , "A H
 A! AF
   AjAÿ q:   !  ("  (AÿÿÿÿqAj"G
   A  A A ï !   Aj6  ( !  Atj" ;  AjA ;  Aj" G
 º@@ AøÿÿÿO
 @   (AÿÿÿÿqAjA  , "A H""M
    (  "  K"Ar"A AK" F
 A! Aj!@@ AK
   ( !  !@  M
  AL
 Av! AtÝC!  (    A H!@ Aj"AÿÿÿÿqE
    At×6@ E
   AtAjâC@ AI
    6   6    A€€€€xr6   Aÿ q: Å6 ð žA÷ÿÿÿ!@@ A÷ÿÿÿ kK
   , !  ( !	@ AòÿÿÿK
 A  j" At"  K"ArAj AI"AL
 	   A H! AtÝC!@ A€€€€xrA€€€€xF
    At×6@   j"	F
   	k"AÿÿÿÿqE
   At"j Atj  j Atj At×6@ Aj"AF
   AtâC   6    A€€€€xr6Å6 ð ä~~A !@  ) "B€€€€T
  B ˆ"§! @@@ §"-  "AUj  A !  Aj" E
A  Aj BQ!A !@ ,  "A€q
 APj"A	K
@ A¯€€€x kA
nL
 A€€€€xAÿÿÿÿ A-FA  Aj  AF!  A
lj!  Aj" 
 A  k  A-F! ƒ~~A !@  ) "B€€€€T
  B ˆ"§!@@@ §"( "AUj  A !  Aj"E
A  Aj BQ!A ! @ ( "Aÿ K
 ’7E
A !@ ( "Aÿ K
  APjA  ’7!@   AÿÿÿÿsA
mL
 A€€€€xAÿÿÿÿ A-FA  Aj AF!   A
lj!  Aj"
 A   k   A-F! ÿ~}# A0k"$ A !@@  ) "B€€€€Z
 A !  B ˆ§!  §!@@  j"-  A G
 Aj"  G
 A !A !    k"A  Aj  I" ! A   !  A jAj"A.:   A
6, Aj ) 7  B…7  B…7  Aj     j Aj ó (! *! A0j$  C     AÄ F  Š~~~~# A0k"$ @ ) "B€ƒP
   F
 @ -  AÀ§j-  AG
 Aj" G
  !@@  G
   A6   6  - !@@@@@ -  "A-F"	
 @ B ƒB€Q
  !
 !
 A+G
 Aj"
 F
 
-  "APj!@ B ƒP
  AÿqA
I
 AÿqA
I
  Aÿq AÿqG
@@ 
 F"E
 A!B !
 
!B !
 
!@@ -  "APjAÿq"A	K
 
B
~ ­Bÿƒ|BP|!
 Aj" G
  ! A
I!  
k!@ B ƒP"
   
F
 AH
  AÿqA0F
 ¬!B !@@ 
  -   AÿqG
 @@  Aj"kAN
  ! !@ )  "BÆŒ™²äÈ‘£Æ | BÐŸ¿þüùóçO|"„B€‚„ˆ À€ƒPE
 B
~ Bˆ|"BˆBÿ€€ðƒB€€€€â	~ Bÿ€€ðƒBä€€€€ÈÐ~|B ˆ 
B€Â×/~|!
  Aj"kAJ
 @  F
    kj!@ -  APj"AÿqA	K
 
B
~ ­Bÿƒ|!
 Aj" G
  !  k!   k¬"}!A!A !A !A ! !@@ 
   PqE
 P
@@ BƒP
   F
  -  "A rAå F
@ BÀ ƒP
   F
 @ -  "AUj"AK
 A tA…€€q
 Aä F
 BƒBQ
 !@@ A¼j"     Aj!A !@  F
 @ -  "A-G
 A! Aj!  A+Fj!@  F
  -  APjAÿqA	K
 B !@@@ -  APj"AÿqA	M
  ! B
~ ­Bÿƒ|  B€€€€S! Aj" G
  !B  }  " |! BƒPE
@ BƒP
   A6   6       ÷B !A !@ BS
 @ 
  Aÿq! 
!@@@ -  "A0F"
   G
  ­}! Aj" G
  BS
B !
@@  
F
  
 j! 
!@@ Aj! 
B
~ 0  |BP|"
Bÿÿ»ºÖ­ð
V
 !  G
  
Bÿÿ»ºÖ­ð
V
@@ 
  !  j! !@ Aj! 
B
~ 0  |BP|"
Bÿÿ»ºÖ­ð
V
 !  G
  !   k¬|!A! B 7(  6$  6   6  
6  :  A:   	:   6  
7  7     ø A0j$ ”~# A0k"$   A 6  B 7 @@@ ( "
  B 7  (! B 7  E
   6$  Aj"6 A !@@   ¯7"E
  k!@@ 
  B 7@ Aj I
  B 7  6  6@@   (O
   )"7(  7  AjýAj! ( !   Ajõ!   6  ($" Asj"A  A G Aj" Iq""	6$   jA  "6  	
    (O
   ) "7(  7  AjýAj!   A jö!   6 A0j$ –~# Ak"$ @@  (  ( "kAu"Aj"A€€€€O
 @@  ( k"Au"   KAÿÿÿÿ AüÿÿÿI"
 A ! A€€€€O
 AtÝC!  ) "7   7  Atj!  Atj ý"Aj!	@  ("  ( "F
 @ A|j"A 6  A|j"( ! A 6  ( !  6 @ E
  ^  G
   (!  ( !   	6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ^  G
 @ E
    kâC Aj$  	Å6 ð –~# Ak"$ @@  (  ( "kAu"Aj"A€€€€O
 @@  ( k"Au"   KAÿÿÿÿ AüÿÿÿI"
 A ! A€€€€O
 AtÝC!  ) "7   7  Atj!  Atj ý"Aj!	@  ("  ( "F
 @ A|j"A 6  A|j"( ! A 6  ( !  6 @ E
  ^  G
   (!  ( !   	6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ^  G
 @ E
    kâC Aj$  	Å6 ð ü  A 6   6 @@ -  "A-F
  B€ƒP
 A+G
 Aj!@@  k"AH
 @@ -  A rA—j  Aj-  AßqAÁ G
 Aj-  AßqAÎ G
   Aj"6  C  ÀÿC  À A-F8   F
 -  A(G
 Aj" F
@ -  "A)F
 @@@ AßqA¿jAÿqAI
  AÿqAß G AFjAÿqAöIq
 Aj" G
 Aj" F
 -  "A)G
    Aj6  Aj-  AßqAÎ G
  Aj-  AßqAÆ G
 @@ AI
  Aj-  AßqAÉ G
  Aj-  AßqAÎ G
  Aj-  AßqAÉ G
  Aj-  AßqAÔ G
 A! Aj-  AßqAÙ F
A!    j6  C  €ÿC  € A-F8   A6ý
~}~~~~~~~~~~~# A0k"$ A !  A 6   (6 @@ ) "Bu|BkT
  - 
 @A *´µk"C  €?’C  €? “\
  )"B€€€V
  µ"8  ) "§!@@ BU
  AÀ© Atk* •! AtAÀ©j*  ”!  8  - AG
  Œ8  B S
  )" §"AtAð©j) V
 @ B R
  C   €C     - 8   AtAÀ©j*  µ”"8  - AG
  Œ8  )!	B !@ B@S
  	P
 Aÿ! B&U
  	y"
§!@ §"AtA°Õj) "Bÿÿÿÿƒ"
 	 
†"B ˆ"
~" B ˆ" Bÿÿÿÿƒ"~|"B ˆ  
~|  T­B †| B †" 
 ~|"
 T­|"BÿÿÿÿÿƒBÿÿÿÿÿR
   AtAtA¸Õj) "Bÿÿÿÿƒ" 
~" B ˆ" ~|"B ˆ  
~|  T­B †| B †  ~B…V­|"
 
|"
 
T­|!A !  B?ˆ"B&„"ˆ!
@ Aê¤
lAu k §jA¾j"A J
 A k"A?K
 
 ­ˆ"Bƒ"
 |BÿÿÿV! 
§ §jAv­!  
Büÿÿƒ 
 
 † Q 
 
BƒBQ 
 
BT 
 B|BT"Bƒ"
 |BÿÿÿV"j"Aÿ AÿI"!B  
§" §"q  sAvj­Bÿÿÿƒ B  !@ - AG
 B !
A !@ B@S
  	B|"P
 Aÿ! B&U
  y"§!@ §"AtA°Õj) "
Bÿÿÿÿƒ"  †"B ˆ"~" 
B ˆ" Bÿÿÿÿƒ"
~|"B ˆ  ~|  T­B †| B †"  
~|" T­|"BÿÿÿÿÿƒBÿÿÿÿÿR
   AtAtA¸Õj) "Bÿÿÿÿƒ" ~" B ˆ" 
~|"B ˆ  ~|  T­B †| B †  
~B…V­|" |" T­|!A !  B?ˆ"
B&„"ˆ!@ Aê¤
lAu k 
§jA¾j"A J
 A k"A?K
  ­ˆ"
Bƒ" 
|BÿÿÿV! § 
§jAv­!
  Büÿÿƒ   † Q  BƒBQ  BT  B|BT"
Bƒ" 
|BÿÿÿV"j"Aÿ AÿI"!B  §" 
§"q  sAvj­Bÿÿÿƒ B  !
@  
R
   F
 	y"§!@ §"AtA°Õj) "Bÿÿÿÿƒ"
 	 †"	B ˆ"~" B ˆ" 	Bÿÿÿÿƒ"	~|"B ˆ  ~|  T­B †| B †" 
 	~|"
 T­|"BÿÿÿÿÿƒBÿÿÿÿÿR
   
BÿÿÿÿoB AtAtA¸Õj) "Bÿÿÿÿƒ" ~"
 B ˆ" 	~|" 
T B ˆ  ~|} B †  	~B…V­}V­|! B?ˆ"§ k Aê¤
lAuj"A–~j!  B…†! AéþJ
  AjAj" 6  Aj ) 7   7  7  A j  ù ((! ) !  At §r - Atr6 @ P Eq )B Rq
  AÿG
  AÄ 6 A0j$ Ò~~~~# Ak"$   (A€€j6 ( !@@ )"BÎ V
  !@ Aj! BÿÁ×/V! BÎ €"! 
 @@ Bã V
  !@ Aj! BÎ V! Bä €"! 
 @ B	X
 @ Aj! Bã V! B
€! 
  A 6ˆ AjA AøØ6 Aj Aò  Aˆjú@@  (ˆkAj"A H
  Aj û Aj Ajü!@@ /„"
 AÖ ! At At AjjA|j5 B †y§kAÖ j!  - ": Ž  :    A(j"6 B(ˆ!B!@ Bÿÿÿÿÿƒ"	B€€€€€V
 @ 	B€€€€€R
  - 
 Bÿÿÿÿÿ?ƒB€€€€€0Q­!@@  |"B€€€T
    A)j"6B ! Bÿÿÿƒ!   7  AÿI
  B 7   Aÿ6 Aj Aj) 7   ) 7    Aj  ý Aj$ ‰
~~~ A 6  (" ("j!@ AH
 @ )  B°àÀƒ†Œ˜0R
  Aj"kAJ
 @@@  F
    kj!@@ -  A0G
 Aj" G
  !  F
 @ ( !A !@@  kAN
 A !@  kAO
 A !A! )  !	  Aj"6  	BÐŸ¿þüùóçO|"	B
~ 	Bˆ|"	BˆBÿ€€ðƒB€€€€â	~ 	Bÿ€€ðƒBä€€€€ÈÐ~|B ˆ§! Aj!@@  G
  !
 !
  O
 @ ,  !  Aj"6  Aj!
 Aj!  A
ljAPj! AK
  F
 
!  I
   /ô!@  G
 @@@@ AÿÿqE
  
AtA€üj5 !B !	A !@   Atj"  5 ~ 	|"
>  
B ˆ!	 Aj"  /ô"I
 @ Aü K
  
B€€€€T
    Atj 	>     /ôAj";ô E
 Aÿÿq
A ! E
A !    ( " j"6   O
A!@@   /ô"O
   Atj" ( Aj"6  Aj! E
   Aü K
A!   Atj 6     /ôAj;ô@@@@@@  kAL
 @@ )  B°àÀƒ†Œ˜0Q
 A!  Aj"kAJ
   F
@ -  A0G"
 Aj" G
  ( E
 A j! ( E
 A j!A ! ) "	§" 	B ˆ§"j!@ AL
 @ )  B°àÀƒ†Œ˜0R
  Aj"kAJ
   F
 @ -  A0G
 Aj" G
  E
A !@  /ôE
 B !	A !@   Atj" 5 B
~ 	|"
>  
B ˆ!	 Aj"  /ô"I
 @ Aü K
  
B€€€€T
    Atj 	>     /ôAj";ô AÿÿqE
 A!    ( Aj"6  
@@   /ô"O
   Atj" ( Aj"6  Aj! E
   Aü K
   AtjA6     /ôAj;ô@@@@ AÿÿqE
  
AtA€üj5 !B !	A !@   Atj"  5 ~ 	|"
>  
B ˆ!	 Aj"  /ô"I
 @ Aü K
  
B€€€€T
    Atj 	>     /ôAj";ô E
 Aÿÿq
A ! E
A !    ( " j"6 A!  O
@@   /ô"O
   Atj" ( Aj"6  Aj! E
   Aü K
A!   Atj 6     /ôAj;ô  G
  ( "E
   ($"j!@ ( 
 @ AH
 @ )  B°àÀƒ†Œ˜0R
  Aj"kAJ
   F
   kj!@ -  A0G
 Aj" G
  !  F
 @ ( !A !@@  kAN
 A !@  kAO
 A !A! )  !	  Aj"6  	BÐŸ¿þüùóçO|"	B
~ 	Bˆ|"	BˆBÿ€€ðƒB€€€€â	~ 	Bÿ€€ðƒBä€€€€ÈÐ~|B ˆ§! Aj!@@  G
  !
 !
  O
 @ ,  !  Aj"6  Aj!
 Aj!  A
ljAPj! AK
  F
 
!  I
   /ô!@  G
 @@@@ AÿÿqE
  
AtA€üj5 !B !	A !@   Atj"  5 ~ 	|"
>  
B ˆ!	 Aj"  /ô"I
 @ Aü K
  
B€€€€T
    Atj 	>     /ôAj";ô E
 Aÿÿq
A ! E
A !    ( " j"6   O
A!@@   /ô"O
   Atj" ( Aj"6  Aj! E
   Aü K
A!   Atj 6     /ôAj;ô@@  kAL
 @ )  B°àÀƒ†Œ˜0R
  Aj"kAJ
   F
@ -  A0G
 Aj" F
  A !@  /ôE
 B !	A !@   Atj" 5 B
~ 	|"
>  
B ˆ!	 Aj"  /ô"I
 @ Aü K
  
B€€€€T
    Atj 	>     /ôAj";ô AÿÿqE
 A!    ( Aj"6  
@@   /ô"O
   Atj" ( Aj"6  Aj! 
   Aü K
   AtjA6     /ôAj;ô@@@@ AÿÿqE
  
AtA€üj5 !B !	A !@   Atj"  5 ~ 	|"
>  
B ˆ!	 Aj"  /ô"I
 @ Aü K
  
B€€€€T
    Atj 	>     /ôAj";ô E
 Aÿÿq
A ! E
A !    ( " j"6 A!  O
@@   /ô"O
   Atj" ( Aj"6  Aj! E
   Aü K
A!   Atj 6     /ôAj;ô  G
   ( Aj6 œA !@   þE
 @ Aq"E
   /ôE
 A  k!A !A !@   Atj"  v ( " tr6  Aj"  /ô"I
 @  v"E
  Aü K
    Atj 6     /ôAj;ôA ! 
@ A I
  Av"  /ô"j!@ E
  Aý K
    At"j   At×6  A  Ø6" /ô j;ôA ! Aý K
A! Š~~@@@@  /ô"  A :  B  At  jA|j( ! A :   ­ gA j­† At  jAxj) ! A :    y†  At  j"Atj) " A|j( "g"A j­"†B R":   ­ †! A  k­ˆ!A !@  /ô"AI
 A!@   As jAtj( "A G! 
 Aj" G
    r:    „ì~# A€k"$  ) !@@ ("AjAWJ
 B  A k"AÀ  AÀ H­ˆ A?J"BÿÿÿV! A(j"Aÿ AÿH"! B(ˆBÿÿÿƒB  !@@ At §rA€€€üq"
  Bÿÿÿƒ!Aê~! AvAé~j! BÿÿÿƒB€€€„! AjA AðØ6 A 6 A;ü  §AtAr6  k!@ E
  AjA  kþ@@ AH
 @@@ Aq"	E
  /üE
 A  	k!
A !A !@ Aj Atj"  
v ( " 	tr6  Aj" /ü"I
   
v"E
 Aü K
 Aj Atj 6   /üAj;ü A I
 /ü! 
 AM
 E
 Av" jAý K
 Aj At"j Aj At×6 AjA  Ø6  /ü j;ü AJ
 A !@@@A  k"Aq"	E
  /ôE
 A  	k!
A !@  Atj"  
v ( " 	tr6  Aj" /ô"I
   
v"E
 Aü K
  Atj 6   /ôAj;ô A I
 /ô! A I
 
 E
  Av" jAý K
   At"j  At×6 A  Ø6" /ô j;ôA !A!
@ /ô" /ü"K
 A !@  I
 @ E! E
  Aj"At"j( "	 Aj j( "K
 	 O
 A !
   ) 7   Aj Aj) "7 @@ §"AjAWJ
   B   ) A k"AÀ  AÀ H­ˆ A?J" 
  §qr­Bƒ|"7    BÿÿÿV6   A(j"6@@  ) "B(ˆ 
  B€€€€€ ƒB(ˆ§qr­|"B€€€T
    A)j"6B ! Bÿÿÿƒ!   7  AÿH
   B 7   Aÿ6 A€j$ Æ~~~# Ak"$ @@@@ A‡I
 @ A
6 A ý6  )7    ÿE
 Aù~j"A†K
 @ A
I
   /ô!@ Aÿÿq!B !A !@ E
 A !@   Atj" 5 B•ç‰Æ~ |">  B ˆ! Aj"  /ô"I
 @ Aü K
  B€€€€T
    Atj >     /ôAj";ô BÿÿÿÿV
 Asj"AK
  E
  /ôE
 AtAÐýj5 !B !A !@   Atj"  5 ~ |">  B ˆ! Aj"  /ô"I
 @ Aü K
  B€€€€T
    Atj >     /ôAj;ô BÿÿÿÿX
A !A! Aj$  ”	~~~# Aðk"$   /ô!A ! A ;ì@ Aý K
 @@ 
 A ! Aøj   AtAüÿqÖ6 /ì!   j";ì@@@ ("E
  ( !@@ 
 A ! 5 !B !A !@   Atj" 5  ~ |"	>  	B ˆ! Aj"  /ô"I
 @ Aü K
  	B€€€€T
    Atj >     /ôAj";ôA ! 	BÿÿÿÿV
 AF
  A AK!
 AtAüÿq! Aÿÿq"Aý K!
A!@@  Atj( "E
  A ;ô 

A !A !@ E
   Aøj Ö6/ô!   j";ô@ AÿÿqE
  ­!B !A !@  Atj" 5  ~ |"	>  	B ˆ! Aj" /ô"I
 @ Aü K
  	B€€€€T
   Atj >   /ôAj";ô 	BÿÿÿÿV
 Aÿÿq!@@  /ô" I
   k O
  j"Aý K
@  M
    AtjA   kAtØ6   ;ô E
  Aq!A !A !@ AF
  Aþÿq!A !A !A !@    jAtj" ( "  Atj( j"Aj"  Aq6    Ar" jAtj" ( "  Atj( j"Aj"   I  Eqr"Aq6   I  Eqr! Aj! Aj" G
 @ E
     jAtj" ( "  Atj( j"Aj"  6   I  Eqr! AqE
   j!@@   /ô"O
   Atj" ( Aj"6  Aj! E
   Aü K
   AtjA6     /ôAj;ô Aj" 
G
   /ô!A! AÿÿqE
@ AÿÿqAt  jA|j( 
   Aj";ô Aÿÿq
  A ! Aðj$   A yA!A!@  Av"Atj"Aj  -    I"!  Asj  "
 Aÿÿ!@ AŒ‘F
 Aÿÿ! -    G
  /! Aÿÿq¿A !@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  AÐ
J
 @  Aá	J
 @  AÑJ
   E
  A*F
  AµG
 A„‘!@  AÜxj                  AÒF
  AêG
Aü!A!  Ažvj	  Að±jR	

A”!A˜!Aœ!A !A¤!A¨!A¬!A°!A´!A¸!A¼!AÀ!AÄ!AÈ!AÌ!
AÐ!  AÑ
G
AÔ!AØ!
AÜ!	Aà!Aä!Aè!Aì!Að!Aô!Aø!A€‘!Aˆ‘! -  ! AÿqžA !@@  €                                 ! Aÿq   A€s" A	IAÃ  vq–@ ("
 A  Aq! ( ! (! ( !@@ AG
 A !A ! A~q!A !A !A !@@  Atj"	( "
AÿK
 @  O
   j 
:   Aj!@ 	Aj( "	AÿK
 @  O
   j 	:   Aj! Aj! Aj" G
 @ E
   Atj( "AÿK
 @  O
   j :   Aj! Ì@ ("E
  Aq! ( ! ( ! (!A !@ AF
  A~q!A !A !@@  O
   Atj  j-  6 @ Ar"	 O
   	Atj  	j-  6  Aj! Aj" G
  E
   O
   Atj  j-  6  +@@ 
 A ! ¡8!   6   6       B 7 @ (E
    ) 7       6   6   r# A°k"$    6 A jA AŒ‘ Ajû7@@ A j¡8" 
 A !    6¬  A j6¨  )¨7 Ajc!  A°j$   Z# Ak"$   A 6 @ E
   6  6  )7  c!  ( !   6  E
  e Aj$   =  A 6 Ab!  ( !   6 @ E
  e  ( !  6  s# Ak"$ @@ 
   A 6  ¡8!  A 6  E
   6  6  )7  c!  ( !   6  E
  e Aj$   q~# Ak"$   A 6 @ (E
   ) ">  B ˆ§Aÿÿÿÿq6  )7  c!  ( !   6  E
  e Aj$   ×~# A k"$   A 6 @ (" ("j" I
 @ E
  b!  ( !   6 @ E
  e  ( !  ) ">  B ˆ§Aÿÿÿÿq6  )7  Ajd  ( !  ) ">  B ˆ§Aÿÿÿÿq6  )7    g A j$    F@  ( " ( "F
 @ E
   ( Aj6   ( !   6  E
  e  6@  (  ( "F
  A 6   ( !   6  E
  e   @ E
     ¡8‚  )# Ak"$   6   AjA‚ Aj$     @ ( "E
    Aj (‚  @ ("E
    (  ‚  † ( !@@  ( " 
 AA  !@ 
 A!@ ("  ("  I"E
   Aj Aj ¦8"
@  G
 A !AA  I! Avz ( !@  ( " 
 AA  @ 
 A@@ ("  ("  I"E
   Aj Aj ¦8"
@  G
 A AA  I! Á# Ak"$ A ! A 6A !@  ( "E
  (! Aj Aj j  ( "Aj"A  ! @ E
   (Atj!@   F
 @@@ ("
 A ! (! Aj   ( Àm  Aj"  G
  (!  Aj$   ¾~# Aà k"$ A !@@  ( "
 A !A ! Aj! (Aÿÿÿÿq!  6\  6X AÌ j AØ jˆ B 7@ B 7  )L7 @A  A j Aj…"E
  A 6T A8j AÔ j j@@  ( " 
 A !A !   Aj!  (Aÿÿÿÿq!    6\  6X A0j AØ jˆ  )8"7(  )07  7A  Aj Aj… AÔ j k (T! Aà j$  v# A k"$ @@  ( " 
 A !A !   Aj!  (Aÿÿÿÿq!    6  6 Aj Ajˆ  )7 Ajì!  A j$   ¡~# A0k"$ A !@@  ( " 
 A !A !   Aj!  (Aÿÿÿÿq!    6$  6  Aj A jˆ  )7 A j Ají A 6, Aj A,j ($ , +"   A HAtAjj )"B ˆ§! §!@@ ($ , +"   A H" "AÿÿÿÿqE
  (  A j  "  Atj!@  O
  j  / ":   Ar" O
  j Av:   Aj!  Aj"  G
   O
   jA :   Ar"  O
    jA :   A,j Ajk@ , +AJ
  (  ((AtâC (,! A0j$   ÷	~~# A k"$ A ! A 6@@  ( "
 A! (AtAj! Aj Aj j )"B ˆ!@@  ( " 
 A !   Aj!  (Aÿÿÿÿq!  §! §!   6  6  AjˆA !@@ ("AÿÿÿÿqE
  ( "  Atj!@@  ( "A€€|jA€€À I
   O
  j :   Aj"	 O
  	j Av:   Aj!  Aj"  G
   O
   jA :   Aj"  O
    jA :   Aj Ajk (!  A j$    ž# A°k"$    ( " 6¬@  E
     ( Aj6  A¤jA˜‘‡ AœjA ‘‡  )¤7P  )œ7H A¬j AÐ j AÈ j€ A”jA¸‘‡ AŒjAÀ‘‡  )”7@  )Œ78 A¬j AÀ j A8j€ A„jAÔ‘‡ Aü jAÜ‘‡  )„70  )|7( A¬j A0j A(j€ Aô jAð‘‡ Aì jAø‘‡  )t7   )l7 A¬j A j Aj€ Aä jA”’‡ AÜ jAœ’‡  )d7  )\7 A¬j Aj Aj€ (¬!  A°j$   þ# A k"$ @@  ( " 
 A !   (!@ 
   G
     ( Aj6   Aÿÿÿÿq6   Aj6 Aj Ajˆ@@ ("E
   (" O
  Aj"  O
   j  I
A !  Aj  Atj ‰@ (" 
 A !   (6   Aÿÿÿÿq6  )7  c!  A j$   Û# A k"$ @@  ( " 
 A ! @   ("G
     ( Aj6   Aÿÿÿÿq6   Aj6 Aj Ajˆ@@ (" E
  Aj (I
A !  Aj   ‰@ (" 
 A !   (6   Aÿÿÿÿq6  )7  c!  A j$   ý# A k"$ @@  ( " 
 A ! @  (" G
     ( Aj6   Aÿÿÿÿq6   Aj6 Aj Ajˆ@@ ("E
   k" (" O
  Aj  O
  Aj  I
A !  Aj  Atj ‰@ (" 
 A !   (6   Aÿÿÿÿq6  )7  c!  A j$   -@  ( "E
  ("E
    z  ( Aj›-@  ( "E
  ("E
    z  ( Ajœ“# Ak"$  A 6 Aj Aj  ("x@ E
   ( "  j!@  -  Aÿ q!@@ ("
 A ! (! Aj  {  Aj"  G
  (!  Aj$   ‰# Ak"$  A 6 Aj Aj  ("x@ E
   ( "  j!@@@ ("
 A ! (! Aj   -  {  Aj"  G
  (!  Aj$   ¯~# AÐ k"$   ) ! B 78  7@  7 B 7A ! @A  Aj Aj†"E
  A 6L A0j AÌ j x  7(  7  )0"7   7 A  Aj † AÌ j y (L!  AÐ j$   Ê~# Ak"$   ) !A ! A 6@ B€€€€T
  §"  B ˆ§j!A !@  ,  "Aÿq!@@ A H
   6 Aj AjA‚A !@ A¿K
 @ AN
 A ! A?q At"r! Aj"
@ AÿÿÃ M
 A !  6 Aj AjA‚A !@ A_K
  Aq!A!@ AoK
  Aq!A! Aq  AxI"!AA  !  Aj"  G
  (! Aj$  ò# Ak"$ @@@  ("
 A ! A ! A 6 Aj Aj Avx@ AF
   ( !A !A !@ "  (O
 (  Atj  j/  6   Aj! Aj! Aj!  I
    (O
A ! (!A !@@@  Atj( "A€xqA€°G
    O
   Aj"Atj( "A€xqA€¸G
    K
 A
tA€ø?q AÿqrA€€j! !   K
  Atj 6  Aj!   I! Aj! 
  Aj y (!  Aj$    €# Ak"$ @@@  ("
 A ! A ! A 6 Aj Aj Avx@ AF
   ( !A !A !@ "  (O
 (  Atj  j"-  At Aj-  r6   Aj! Aj! Aj!  I
    (O
A ! (!A !@@@  Atj( "A€xqA€°G
    O
   Aj"Atj( "A€xqA€¸G
    K
 A
tA€ø?q AÿqrA€€j! !   K
  Atj 6  Aj!   I! Aj! 
  Aj y (!  Aj$    8 @  ( " E
 @ 
 A  Aj ž@ 
 A AA  ( J# Ak"$ @@  ( " 
 A !  Aj  Aj‡  )7  ñ!  Aj$   )  Aj  AqA¸’j-  :     AvA¸’j-  :  Z  Aj  AqA¸’j-  :     AvA¸’j-  :   Aj  AðqAvA¸’j-  :   Aj  AvAqA¸’j-  :  ©@@ AÿÿK
  Aj AqA¸’j-  :    AvA¸’j-  :  A! Aj AvAqA¸’j-  :   Aj AvAqA¸’j-  :   AÄ :   AjAÄ :   Aj AqA¸’j-  :   Aj AvAqA¸’j-  :  A! Aj AvAqAÄ’j-  :   Aj A
vAqA¸’j-  :   Aj A€€<j"AvAqA¸’j-  :   Aj AvAqAÀ’j-  :     6   6  A Ù6   Ÿ7!~A (¸µk# !@  E
    7     A (¼µk Ó	# Ak"$ @@@  ("  ("G
 A!  k"Au"A AK!A !A !@@   ¢"E
 ë! (!	 E
 	E
  	Aj"	6@ 	
   ( (   Aj" G
 A! AF
 Aj"E
A !@    §6Aü—AÐ– Aj³! (!	 A 6@ 	E
  	^ Að—G! Að—F
 Aj" G
   	E
  	Aj"6 
   ( (   Aj$   ¨@  ‰E
  @  Aj"‰
   A j"‰
   A0j"‰
   AÀ j"‰
   AÐ j"‰
   Aà j"‰
   Að j"‰
   A€j"‰
  Aj!  A j  ‰! ‚	# A0k"$ @@@ ("E
  ( !A !A !@@@@@  j-  "Aú G
  Aj! AŠjAÿqAªK
  Awj"AK
A tA“€€qE
 Aj" G
  ! 
  A 6  B 7   AjA Ã ("E
  6@@ AÿÿÿÿK
   kAn" jAtAj" AtAj"O
 A 6 B 7   AjAÃ ("E
  6@@ AL
 A    I!A !	 AVA  Ø6"
 j!A !A ! 
!@@ "Aj!@@  ( j-  "
Awj"AK
 A tA“€€q
@@@ 
Aú G
  AM
A ! A 6   Aj! A|j! 
AŠjAÿqA«I
 	AÕ l 
jA_j!	 AI
 E
  	Av:   AF
 Aj 	Av:   AF
 Aj 	Av:   AF
 Aj 	:   Aj! A|j!A !A !	 Aj!  ("I
 @ E
 @@ AI
  Aj! 	AÕ lAÔ j!	@ AF
  	AÕ lAÔ j!	 AF
  	AÕ lAÔ j!	 AF
  	AÕ lAÔ j!	 Aj"E
A !@ E
  	A Atkv:   Aj! Aj! Aj" G
  (!@  O
   (  j-  A>Fj!  k!@@  O
  AL
 At"   KAÿÿÿÿ AÿÿÿÿI"AV" j!  j"A A  kØ6 k!@ Aj Aj-  :   A~j A~j-  :   A}j A}j-  :   A|j" A|j"-  :    
G
  
E
 
J 
 j   K! 
! !  6  6  6     Ã ( "E
  6Å6   A 6, B 7$   A$jA Ã ($"E
  6( J A0j$ ú
# A k"$ @@@@@ ("E
  ( !A !@@  j-  A>F
 Aj" G
  ! Av"Aj"AJ
 A 6 B 7   AjA Ã ("E
  6A ! AVA  Ø6" j!	A!
 ! !@@@@@@ (  j-  "
Awj"AK
 A tA“€€q
 
A>F
 
À"A€q
  “7E
 AIAP 
A`j 
 AŸjAI"ÀA9J j!@@ 
AqE
  E
  At:   E
  -   j:   Aj! Aj! 
As!
 (! Aj" I
    Aj!@@  k 
AsAqj" M
  AL
A ! At"   KAÿÿÿÿ AüÿÿÿI"AV"
 j"A   kØ6@ Aq"
E
 @ Aj" 	Aj"	-  :   Aj" 
G
 @ AI
 @ Aj 	Aj-  :   A~j 	A~j-  :   A}j 	A}j-  :   A|j" 	A|j"	-  :   	 G
  
 j!	 
 j! E
 J 	  j  I! !
  	6  6  
6   Aj Ã ("E
  6 J A j$ Å6 ø# A0k"$ @@@@@ ("
 A!A !A ! ( !A !A !	@@  	j-  "A€F
@@@ ÀA H
   jAj"
 O
 A 6, B 7$   A$jAÃ ($"E
  6(  kAj"
 O!A! 
! 
 A 6  B 7   AjAÃ ("E
  6 Aj! 
!  	j"	 I
 @ A€€€
I
  A 6 B 7   AjAÃ ("E
  6A !A !@ E
  AV"A  Ø6 j!  k! ( !A !A !@@  j"-  "
A€F
@@ 
ÀA H
  
Aj!@@  Asj"	 
M
  !	  	 j"
I
  	k"  
kK
  
jA  Ø6  O
 	  Aj"
kK
  I
  k 	I
@ 	E
   j  
j 	×6 -  "	Aj!
 	Aj!	A !@ Aj"	 O
   	j-  !  I
A 
k"	  kK
  j  	Ø6A!
 	 j! 
 j" I
  Aj!  6  6  6        IÃ ( "E
  6 J A0j$  ý~# Að k"$ @@ 
 AÀ
!A !A !A !A !A !	 A6l Aáÿ6h  )h70  A0jö! A	6d AÌÐ6`  )`7(  A(jö! A6\ Aß°6X  )X7   A jö! A6T AÉ’6P  )P7  Ajö!
 A6L Aê•6H  )H7  AjAÀ
÷! A6D AÜ’6@  )@7A   Ajö" AÿÿJ!	 A G! A G! 
A G!   ) "7   78         	Ï! Að j$  Õ~# AÐ k"$ @@@ 
 A !A !A !	A ! A	6L Aêž6H  )H7   A jö! A6D AÇ”6@  )@7  AjA÷! A6< Aƒ‹68  )87  AjA÷!	 A64 Aê•60  )07A !
 	 r  AjA÷"rA H
 ¬ ¬~"B€€€€|BÿÿÿÿV
  	¬~"B€€€€|BÿÿÿÿV
 §AøÿÿÿJ
   ) "7   7(        	 Ú!
 AÐ j$  
·~# Að k"$ @@@ 
 A!A !A !A !	A ! A	6l Aêž6h  )h7(  A(jö! A6d A¯Ö6`  )`7   A jA÷! A6\ AÇ”6X  )X7  AjA÷! A6T Aƒ‹6P  )P7  AjA÷!	 A6L Aê•6H  )H7@ 	 r  AjA÷"rA H
  ¬ ¬~"
B€€€€|BÿÿÿÿV
  
 	¬~"
B€€€€|BÿÿÿÿV
  
§AùÿÿÿN
  A G! A 6D B 7<   A<jAÃ (<"E
  6@ J  ) "
7   
70        	  Û Að j$ ì	# AÀ k"$  A6< Aµ¥68  )87@@@  Ajï"
   B 7   A:   AjA 6 @@ å
  ë
   A :   A :   A64 Að—60  )07  Ajï!A ! A 6, B 7$@@@@@ å"E
  ²E
@@ 
 A !  (Aj"6 E
@ å"E
   (Aj"6 E
	 ("E
  Aj"6 
   ( (   ( (F
A !@   §6 A !@ E
   ¬!  6@@ ((" (,O
  A 6  ( !	 A 6  ( !
  	6 @ 
E
  
^  (6  Aj6(  A$j A j Aj»6( (! A 6 E
  ("
E
	  
Aj"
6 

   ( (   ( ! A 6 @ E
  ^ Aj" ( (kAuI
     ( ( 6 @ E
  à!  6  A$j A j Aj»6( (! A 6@ E
  ("E
  Aj"6 
   ( (   ( ! A 6  E
 ^A !  A :   E
  ("E
  Aj"6 
   ( (     ($6    ((6   (,6A!   :  E
  ("E
  Aj"6 
   ( (   ("E
  Aj"6 
   ( (   ("E
   Aj"6@ 
   ( (   AÀ j$  é@@@  (  ( "kAu"Aj"A€€€€O
 A !@  ( k"Au"   KAÿÿÿÿ AøÿÿÿI"E
  A€€€€O
 AtÝC! ( ! A 6   Atj" 6  ( ! A 6   6  Atj! Aj!@@  ("  ( "G
  !@ Axj"( ! A 6  Axj" 6  A|j"( ! A 6  A|j 6  ! !  G
   (!  ( !   6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (   Axj"( ! A 6 @ E
  ^  G
 @ E
    kâC Å6 ð |  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    (!  A 6@ E
  ^@  ( "E
    6 J   ~# A€k"$  Aø jA 6  Að jB 7  B 7h@@@@ (" ( "F
   kAu"A AK! As!	 Aj!
 ) "B ˆ§! Aô j!
 §!A !@  ( " At"j( "6d@ E
   ( Aj6  ( !@@  jAj( "
 A !  (Aj"6 E
@ é"E
   (Aj"6 E
 ("E
  Aj"6 
   ( (  A!@@@ Aä jAŸ‰‰
 A    
G"!@@@@ Aä jAéÙ‰
  Aä jA·½‰E
  	r
 
AéÙ (x!  6x@ E
  ("E
  Aj"6 
   ( (     (h6    (l6   (p6 B 7h (t! B 7p   6 (x! A 6xA!  A:    6@@ Aä jAÿÙ‰
  Aä jAÒó‰E
  ­B † ­„"7@  7 AÔ jA Aj  ¹ (\! A 6\ (X! (T! B 7T (`! AÔ jÄ@@ Aä jA“Ú‰
  Aä jA²‰E
  ­B † ­„"78  7 AÔ j Aj´ (\! A 6\ (X! (T! B 7T (`! AÔ jÄ@@ Aä jA»Ù‰
  Aä jAö…‰E
  ­B † ­„"70  7 AÔ j Ajµ (\! A 6\ (X! (T! B 7T (`! AÔ jÄ@@ Aä jAÙÙ‰
  Aä jA±ÿ‰E
@  	r
  
AÙÙ (x!  6x@ E
  ("E
  Aj"6 
   ( (     Aè j¾A!  ­B † ­„"7(  7  AÔ j A j¶ (\! A 6\ (X! (T! B 7T (`! AÔ jÄ@@@ Aä jAõù‰E
 A‰Ú! Aä jA¶„‰E
AÊÙ! Aä j  
 Aä j„ (x!  6x@ E
  ("E
  Aj"6 
   ( (     Aè j¾A!  ­B † ­„"7H  7  AÔ jA    ¹ (\! A 6\ (X! (T! B 7T (`! AÔ jÄ@ AF
 @ (h"E
   6l J  k!  6p  6l  6hA !  A :   A :  A! E
  J !@ E
  ("E
  Aj"6 
   ( (   ! (d! A 6d@ E
  ^@    Aj" G
  Aô ji (x! A 6x@ E
  ("E
  Aj"6 
   ( (     (h6    (l6   (p6 B 7h (t! B 7p   6 (x!  A:    6 A 6x (x! A 6x E
  ("E
  Aj"6 
   ( (   (t! A 6t@ E
  ^@ (h"E
   6l J A€j$  Ï  A 6  B 7    ( 6    (6   (6 A 6 B 7   A 6 (! A 6  (!   6@ E
  ^  A 6 (! A 6  (!   6@@ E
  ("E
  Aj"6 
   ( (    A:    ”	~# AÀ k"$  A 6<@@@@@@  ("AI
 @@@  ( "-  "A‚~j  Aj-  AþG
  A~j60  Aj6,  ),7  Aj§6  A<j A j‘ ( !  A 6   E
  e Aj-  AÿF
@ AF
  AïG
  Aj-  A»G
  Aj-  A¿G
  B 7@ A}j" E
    6  Aj6  )7   ¦6  A<j A j‘ ( ! A ! A 6 @  E
   e@ (<" E
   (! A j A<j xA ! (<" E
  ("E
 ) "§! B ˆ§"Aj!A ! A !@   O
@@   Atj( "AG
    Aj"  K"	Aj!@@   G
  	!    F
   Aj" Atj( AG
    O
  Atj 6  Aj!  Aj"  I
   A j A<j x A j A<j x E
  ( !A ! @   ($O
 (   Atj   j-  AtAÈ’j/ 6   Aj"  G
    A~j68  Aj64  )47  Aj¨6  A<j A j‘ ( !  A 6   E
   eA !A ! @ (<"E
  (!  A j A<j  x (<" E
   ("E
  ) "§! B ˆ§"Aj!A ! A !@   O
@@   Atj( "AG
    Aj"  K"	Aj!@@   G
  	!    F
   Aj" Atj( AG
    O
  Atj 6  Aj!  Aj"  I
  A<j y (<!  AÀ j$    å~# A0k"$   (!A ! A 6, A j A,j j@@@ E
   ( !@  Atj( !A !@@@@@  AtAÈ’j/ G
  !  Ar"AtAÈ’j/ F
  Ar"AtAÈ’j/ F
  Ar"AtAÈ’j/ F
 Aj"A€G
  A,j k  F
 AþÿÿÿK!A ! 
   ) "7  7 A j Ají Aj A,j ($ , +" A HAtAjj )"BÿÿÿÿX
 §"Aþ:   BÿÿÿÿX
 AjAÿ:  A!@ ($ , +" A H""AÿÿÿÿqE
  B ˆ§! (  A j " Atj! @  O
  j / "Av:   Ar" O
  j :   Aj! Aj"  G
  - +! ÀAJ
 (  ((AtâC  ($I
  (  j :   Aj" G
  A ! A,j k (,! A0j$  ø# Ak"$ A ! A 6 Aj Aj  ("Ajj AjA(†@ E
   ( !@@@@@@@  j-  " Avj   AjA‚²… AjAÍ§…  AÜ G
 AjAÜ † Aj  À† Aj" G
  AjA)† (! Aj$  ¥# Ak"$ A ! A 6 Aj Aj  ("AtAjj AjA<†@ E
   ( ! @   j-   Aj« Aj , † Aj , † Aj" G
  AjA>† (! Aj$  (   B 7  Aˆ˜6   AjB 7   AjA 6   '  Aˆ˜6 @  ("E
    6 J  ,  Aˆ˜6 @  ("E
    6 J  AâC@@  ("  ("F
   (" I
    k"K
    kK
  K
   j"I
  I
@  F
   j  j  k×6  ( k!   6 -@ (" ( ("kM
     6   6 -@ (" ( ("kM
     6   6    („@@ (" (" ("k"M
  Aj  kÜ (! (!  j   I! A 6   6   6    (6 A 6 B 7œ~@@  (  ("k" O
    (I
  (" Av "A€ A€K"Aj" j" I
  n­ ­~"B ˆ§
@ §" M
   Aj  kÜ  O
     j6 £~@  (" j" I
 @  (  ("k" O
   (" Av "A€ A€K"Aj" j" I
  n­ ­~"B ˆ§
@ §" M
   Aj  kÜ  O
     j6 …	~@@ ("E
   (" j" I
@  ("  ("k" O
   (" Av "A€ A€K"	Aj" j" I
  	n­ 	­~"
B ˆ§
@ 
§" M
   Aj  kÜ  (!  (!  (!  O
     j"6  k" I
  k I
  j (  ×6    ( j6   ù@@@  óE
     (Aj"6 E
@  ó"E
   (Aj"6 E
  ("E
   Aj"6@ 
     ( (  AÝC" 6 A ; A¸˜6 @  éE
     (Aj"6 E
@  é"E
   (Aj"6 E
  ("E
   Aj"6@ 
     ( (  AÝC!@ 
  A :  A 6 A 6 AÔ˜6  AjA — A 6 ("AF
 A :   6 A 6 AÔ˜6   Aj"6 E
 Aj — A 6 ("E
  Aj"6 
  ( (  @  å
 A !    (Aj"6 E
@  å"E
   (Aj"6 E
  ("E
   Aj"6@ 
     ( (  AÝC!@ 
  A :  A 6 A 6 Að˜6  AjA Á ("AF
 A :   6 A 6 Að˜6   Aj"6 E
 Aj Á ("E
  Aj"6 
   ( (    ("E
    Aj"6@ 
     ( (    :   B 7   6   AjB 7   AjB 7   AjB 7   A$jA 6   ª  AjÒ  (!  A 6@ E
  ^  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    ( !  A 6 @ E
  ("E
  Aj"6 
   ( (     »@@  ("  ("G
   Aj! !  Aj!@   ("AvAüÿÿqj"(  AÿqAtj"   ( j"AvAüÿÿqj(  AÿqAtj"F
 @ ( ! A 6 @ E
   ( (  @ Aj" ( kA€ G
  Aj"( !  G
   (!  (! ! A 6 @  kAu"AM
 @ ( A€ âC    (Aj"6  (" kAu"AK
 A€!@@@ Aj A€!   6@  F
 @ ( A€ âC Aj" G
   ("  ("F
      kAjA|qj6@  ( "E
    ( kâC  €# Ak"$ @@@@  ($"E
   Aj!@  ( "
@@  (   ( jAj"AvAüÿÿqj(  AÿqAtj( " ( ( E
   (  ($  ( jAj"AvAüÿÿqj(  AÿqAtj"( ! A 6 @ E
   ( (      ($"Aj"6$A   ("  ("kAtAj  F   ( jkAjA€I
 A|j( A€ âC    (A|j6  ($!@ - 
   ( (   A: @@@  ( ( E
 A !  ( ( "E
   ( !   6 @ E
  ("E
  Aj"6 
   ( (  @ ("E
   (Aj"6 E
  (!   6@ E
  ("E
  Aj"6 
   ( (  A !@  (éE
  ("E
   ( Aj6  !  6  Aj„ (! A 6@ E
  ^    ($"6 
   ( "
 A ! A 6  Aj Aj„ (! A 6@ E
  ^  A 6  (Aj"6 E
@ Ï"E
 @A   ("  ("kAtAj  F  ($  ( j"G
   AjÔ  (   ($j!  (!  AvAüÿÿqj(  AÿqAtj 6     ($Aj6$  ( !  A 6  Aj$   Ý
# A k"$ @@@  ("A€I
    A€xj6  ("( !   Aj"6@@  ("  (F
  !@   ( "M
   k!   kAuAjA~mAt"j!@  F
    ×6  (!    j"6    j6A  k"Au  F"A€€€€O
 At"ÝC"	 j!
 	 A|qj"!@  F
    kj! !@  ( 6  Aj! Aj" G
    
6   6   6   	6  E
   âC  (!  6     (Aj6@  ("  (k"  ("  ( "k"O
 @  F
  A€ ÝC6   Ajç A€ ÝC6   Ajè  ("( !   Aj"6@@  ("  (F
  !@   ( "M
   k!   kAuAjA~mAt"j!@  F
    ×6  (!    j"6    j6A  k"Au  F"A€€€€O
 At"ÝC"	 j!
 	 A|qj"!@  F
    kj! !@  ( 6  Aj! Aj" G
    
6   6   6   	6  E
   âC  (!  6     (Aj6   Aj6A Au  F"A€€€€O
  At"ÝC"6   j"6   j6  6 A€ ÝC6 Aj Ajé@  ("  ("F
 @ Aj A|j"ê   ("G
   (!  ( !   (6   6   (6  6   (6  6  (!   (6  6@  F
     kAjA|qj6 E
    kâC A j$ ð ·@  ($"E
   (   ( jAj"AvAüÿÿqj(  AÿqAtj"( "- 
  A 6   ( (      ($"Aj6$A   ("  ("kAtAj  F   ( jkAjA€I
  A|j( A€ âC    (A|j6:   B 7   6   AjB 7   AjB 7   AjB 7   A$jA 6   2 @  Ó" E
 @@  (Aj      ( (    S  Aœ˜6   (!  A 6@@ E
  ("E
  Aj"6 
   ( (     X  Aœ˜6   (!  A 6@@ E
  ("E
  Aj"6 
   ( (    AâC    -   - 	qAq   A: 	  (à v  (!  A 6@ E
  ^  Aj˜  Aœ˜6   (!  A 6@@ E
  ("E
  Aj"6 
   ( (     {  (!  A 6@ E
  ^  Aj˜  Aœ˜6   (!  A 6@@ E
  ("E
  Aj"6 
   ( (    AâC 6A !@@  - AG
   ("(E
  ( AjF!  “@@  ("("E
   (Aj"6 E
  (!  Aj Ajƒ@@  ("("E
 @ "( "
  @  ("( G! ! 
    6  @  ("(
     (6\  AjÂ  Aœ˜6   (!  A 6@@ E
  ("E
  Aj"6 
   ( (     a  AjÂ  Aœ˜6   (!  A 6@@ E
  ("E
  Aj"6 
   ( (    AâC 6A !@@  - AG
   ("( E
  ( (F!  D@@  ("( "E
   (Aj"6 E
  (!   Aj6  @  ("( 
     (6Ó@@@  ("  (F
  !@  ("  ( "M
   k!   kAuAjA~mAt"j!@  F
    ×6  (!    j"6    j6A  k"Au  F"A€€€€O
 At"ÝC" j!	  A|qj"!@  F
    kj! !@  ( 6  Aj! Aj" G
    	6   6   6   6  E
   âC  (!  ( 6     (Aj6ð Ý	@@@  ("  ( "F
  !@  ("  ("O
    kAuAjAmAt"j  k"k!@  F
    ×6  (!   6    j6A  k"Au  F"A€€€€O
 At"ÝC"	 j!
 	 AjA|qj"!@  F
    kj! ! !@  ( 6  Aj! Aj" G
    
6   6   6   	6  E
   âC  (! A|j ( 6     (A|j6ð Ó@@@  ("  (F
  !@  ("  ( "M
   k!   kAuAjA~mAt"j!@  F
    ×6  (!    j"6    j6A  k"Au  F"A€€€€O
 At"ÝC" j!	  A|qj"!@  F
    kj! !@  ( 6  Aj! Aj" G
    	6   6   6   6  E
   âC  (!  ( 6     (Aj6ð Ý	@@@  ("  ( "F
  !@  ("  ("O
    kAuAjAmAt"j  k"k!@  F
    ×6  (!   6    j6A  k"Au  F"A€€€€O
 At"ÝC"	 j!
 	 AjA|qj"!@  F
    kj! ! !@  ( 6  Aj! Aj" G
    
6   6   6   	6  E
   âC  (! A|j ( 6     (A|j6ð    A:    6      A:    8   á~}# Ak"$   A :   A 6 @ ("E
 @ ( "A. ¯7E
   ) "7   7 ò!@  - AF
   A:    8 A!A !A!	A !
A!@@@ -  AUj A!	A!
A!A !	A !
A !@  O
 A !@@@  j,  "A€qE
  !@ APj"A	M
  !  Aš³æÌIq!
A !A !@ 
AG
  ­B
~"B€€€€ B€€€€TBþÿÿÿƒ ­|"B€€€€ B€€€€T"§! ! Aj" G
  Aq
 A !@ 	
    6 A   A€€€€xAÿÿÿÿ 
K!@ 
E
   A:   A  k6    6   A:  Aj$   
   - AIy}@@@  -     ( ¯ AÿÿÿÿA   * "C   Ï`!  CÿÿÿN_!@ C   Ï C   Ï^"‹C   O]E
  ¨   A€€€€x   0 @@@@  -    ( ³  ( ²¯   * &   A 6  B 7  AŒ™6   Aj ë  &   A 6  B 7  AŒ™6   Aj ì  M~# Ak"$   A 6  B 7  AŒ™6   ) "7   7  Aj í Aj$      Ô   ÔAâC A¥}@@@  Aj" îE
   ï!AÝC" A 6  B 7  AŒ™6   Aj ë    (Aj"6 
  ð!AÝC" A 6  B 7  AŒ™6   Aj ì    (Aj"6 E
   
   Ajð
   Ajï   q# A k"$ @@ ( "
  B 7 (! B 7 E
   6  Aj6  )7   Aj Ají) 7 A j$ •}# Ak"$ @@  Aj" îE
   ïù!   ð! AÄ©$6< A°©$6 A<j" Aj" Ì: AŒ©$6< Aø¨$6 B€€€€p7„  Ý8" AÔ©$6  A0jB 7  B 7( A68 Aj Ç/ AŒj Aj€! AŒ©$6< Aø¨$6  AÔ©$6 @ , 3AJ
  ((L  Û8 AjA˜©$‹9 Ú8 ( !  Aj$   °# A0k"$  A6, Aø£6(  )(7A !@  Aj°E
 @  ü" E
   (! B 7 @ E
   6$   Aj6   ) 7  Aj°!  ^ B 7 B 7   Aj°! A0j$  ã
~# AÐ k"$ A$ÝC!@@ E
   (Aj"6 E
  Ÿ" (Aj"6 E
   B 7  A :   B 7   6  §  Aj! ™!@@@ 
  E
 A$ÝC!  (Aj"6 E
  Ÿ" (Aj"6 E
 ¤ AÈ j  ( «@@  - "AÿF
 @ 
   )H7  A )è™70 AÇ j  A0j Atj(   )H!  A :    7@@ à"è"
 A !@  ( (, "E
   (Aj"6 E
 ("E
  Aj"6 
   ( (    (!   6@ E
  ("E
  Aj"6 
   ( (   ("E
  Aj"6@ 
   ( (    (! A6@ Aµ¥6<  )<7@  Aj"E
  ("E
  Aj"6 
   ( (   ("E
  Aj"6 E
@  AsrAG
  AÈ j  ( «@@  - "AÿF
 @ 
   )H7  A )è™70 AÇ j  A0j Atj(   )H!  A :    7 à!  (!   6 E
 ("E
  Aj"6 E
 A(j  ( «  )(7 A0j Ajâ@@@@@@@  - "AÿF
 @ AG
  ( "
   (0"6   (4"6   (86 A )è™7H AÇ j  AÈ j Atj(     (0"6   (4"6 (8!  A:    6   6 J   (0"6   (4"6   (86  -    k! E
@@ à"è"
 A !@  ( (, "E
   (Aj"6 E
 ("E
  Aj"6 
   ( (    (!   6@ E
  ("E
  Aj"6 
   ( (   ("E
  Aj"6@ 
   ( (    (! A0jA÷Áü!@  -    (  (k!¯   (! AL
AÝC ñ" (Aj"6 E
@   þ"E
   ("Aj"6 E
  6 
   ( (   ( ! A 6 @ E
  ^  (!@  A0jAµ¥ü" AjAéÙÿþ"E
   ("Aj"6 E
  6 
   ( (   ( ! A 6 @ E
  ^  (! A6$ Að—6   ) 7  Aj"E
 ("E
  Aj"6 
  ( (   ("E
   Aj"6@ 
   ( (   AÐ j$    ¤# Ak"$ AÝC!   ( " 6@  E
     ( Aj6   Aj Aj ü"×"   (Aj"6@ E
  ( ! A 6 @ E
  ^ (! A 6@ E
  ë Aj$    C@@@ -   (!   ("6     k6¯    )7 •# Ak"$   (!  A 6@@ E
  ("E
  Aj"6 
   ( (    (!  A 6@ E
  ("E
  Aj"6 
   ( (  @  - "AÿF
  A )è™7 Aj  Aj Aj Atj(    Aÿ:   ( !  A 6 @ E
  ("E
  Aj"6 
   ( (   Aj$    ¿# A k"$   (!  (! A6 A÷Á6  )7@@    Ajö F
 @  (
 @@  (è"
 A !@  ( (, "E
   (Aj"6 E
 ("E
  Aj"6 
   ( (    (!   6@ E
  ("E
  Aj"6 
   ( (    (!  A 6 E
  ("E
  Aj"6 
   ( (    (! AjA÷Áü! AÝC ñ" (Aj"6 E
@    þ"E
 @ (Aj   ( (    ( !  A 6  E
  ^ A j$    ("  (   ”M   A 6  B 7  B 7  Aø™6    6   A:   AjA 6 @ (E
    A …  ˜# Ak"$   ( ! AjA÷Áü! AÝC ñ" (Aj"6@ E
 @    þ"E
 @ (Aj   ( (    ( !  A 6 @ E
  ^ Aj$  ˆ   A 6  B 7  A 6  B 7  Aø™6    ( 6   (6   (6 A 6 B 7    6   A: @ (
   (  (k"AL
    …   z~# Ak"$   B 7  A :   B 7  Aø™6 A$ÝCá" (Aj"6@ 
     6   ) "7   7   ˆ Aj$   à# Ak"$ @@@@ ("
 A !A !A ! AL
 ( ! AV!@@ Aq"
  ! !A ! ! !@  -  :   Aj! Aj! Aj" G
 @ AI
   j!@  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
   j!  k"AL
@@  - "AÿF
   Aj!@ AG
  ( "E
   6 J A )Øš7 Aj  Aj Atj(    A:    6   6   6   … Aj$ Å6 S  B 7  A :   B 7  Aø™6 A$ÝCá" (Aj"6@ 
     6    Š  ¦# AÀ k"$ @@@@  ( Atj( j"- Aq
  A0j ("B AA ( (  )8B U
 B 7  B 7(   ˆ@@@ (4"AqE
 @ (0" ("O
   60 ! Aj!@ Aq
 A ! A :  Aj! Aj! (!@@  ( "k"AøÿÿÿO
 @ AI
  ArAj"AZ!  A€€€€xr6  6  6  :  Aj! 
A !Å6   ×6  jA :   ( , " A H"! (!@@  ( Atj( j"- AqE
  B78 B 70A! A0j ("B AA ( (  (8!  I
  6$   Aj 6   ) 7   Ajˆ , AJ
  (L AÀ j$  s~  A 6  B 7   6   A :    6  Aø™6 @ (
   ( Alj( j" ( (  "B€€€€|B€€€€Z
    §…   °# Ak"$   A6  ( "(!  A 6 @@ AF
  ("E
  Aj"6 
   ( (  @  - "AÿF
  A )Øš7 Aj  Aj Aj Atj(    Aÿ:   Ô!  Aj$    ±# Ak"$   A6  ( "(!  A 6 @@ AF
  ("E
  Aj"6 
   ( (  @  - "AÿF
  A )Øš7 Aj  Aj Aj Atj(    Aÿ:   ÔA$âC Aj$   A   (    	   A ÙÄ# Ak"$  Aj"! !@@ ("E
 @@   "("O
  ! ( "
   O
 ("
  Aj!AÝC" 6 B 7    6  6 @ ( ( "E
   6  ( ! ( È  (Aj6    (Aj"6@ E
 A$ÝC  Ÿ" (Aj"6 E
  §  à!@@ ( "E
  !@   ( I" !   Atj( "
   F
   (I
 A !@   é"
 A !@  ( (, "E
   (Aj" 6  E
 (" E
   Aj" 6  
   ( (   Aj ®A$ÝC! (! A 6 (!  (! B 7 A 6 B 7  6  A:   6  6   6 Aø™6  (
    k"AL
   … A6@ ("E
   6 J@ E
  ("E
  Aj"6 
   ( (   ("E
   Aj"6@ 
   ( (   Aj$   ë~# A0k"$   ) "7  7(   Ajˆ  ( ! A6$ Aµ¥6   ) 7@@  Aj"E
  ("E
  Aj"6 
   ( (    ( !  A6 Að—6  )7 @   " E
   ("E
   Aj"6 
     ( (   A0j$  ¦# AÀ k"$ @@@@  ( Atj( j"- Aq
  A0j ("B AA ( (  )8B U
 B 7  B 7(   “@@@ (4"AqE
 @ (0" ("O
   60 ! Aj!@ Aq
 A ! A :  Aj! Aj! (!@@  ( "k"AøÿÿÿO
 @ AI
  ArAj"AZ!  A€€€€xr6  6  6  :  Aj! 
A !Å6   ×6  jA :   ( , " A H"! (!@@  ( Atj( j"- AqE
  B78 B 70A! A0j ("B AA ( (  (8!  I
  6$   Aj 6   ) 7   Aj“ , AJ
  (L AÀ j$  1@ ( ( k"AJ
    Aj"  –   …ô# Ak"$ @@  - "AÿF
 @ AG
 @ ( " E
    6  J A 6 B 7   ( 6   (6  (6 A 6 B 7  A )Øš7 Aj   Aj Atj(    A 6  B 7   Aÿ:    ( 6    (6   (6 A 6 B 7   A:  Aj$ ð~# Ak"$ @@@ - 
  (" ( Alj( j" ( (  "B€€€€Z
 @@ PE
 A !A ! §"AL
 AV"A  Ø6 j! - 
@ ("E
   ( Apj( j" (Aj"6 E
  6   k6 ( (!  )7 @@  B   
   A 6  B 7    6   6   6 A !  ( Apj( j"(" E
    Aj" 6@  
   ( (  @ E
  J Aj$  Å6 ¯ Z~@@@@  -    ("   ( Alj( j"   ( (  "B€€€€Z
 §¯   (  (k @# Ak"$   ( !  A6 Aµ¥6  )7    ˆ!  Aj$   ›# Ak"$     (Aj"6@ E
 A$ÝC  Ÿ"   (Aj"6 E
   ¤ Aj  «  )7  ¿!  ("E
    Aj"6@ 
     ( (   Aj$   ‰~# Ak"$   à! A6x Aíð6t  )t70A !@  A0jÒE
  A6` A¹Î6\  )\7(   A(jô6h Aè jA´ÿ‰! (h! A 6h E
  ^@@ E
  ("E
  Aj"6 
   ( (      (Aj"6 E
  Aè j Aô j   As"þ"€A ! @@  A Gq
  (l!  )h"7P  7  AÜ j  A j¿  (\" 6h  (`  k"6l  ‚A !@   ƒE
  A6L A¶™6H  )H7  Aj°E
   )h"7@ ( ( !  7  Aj  E
  A6< AŸ´68  )87  Aj°!@  E
   J  Aj$   2@ - AF
 ¯  (!   ("6     k6T ( ! A 6 @@ E
   ( Apj( j"("E
  Aj"6 
   ( (   @ ( "E
   6 J0   A :    B 7   6  A 6  Aèš6   B 7  ò# Ak"$ @  -  "AÿF
  A )ðš7 Aj  Aj Aj Atj(    Aÿ:    (!  A 6@@ E
  ("E
  Aj"6 
   ( (    (!  A 6@ E
  ("E
  Aj"6 
   ( (    (!  A 6@ E
  ^ Aj$        A$âC“~# A k"$ @  (˜E
 @  ("- AG
  Aj œ@  -  "AÿF
   Aj!@ 
   )7  A )ðš7 Aj  Aj Atj(   )!  A :     7 Aj —@ (" ("F
 @  -  "AÿF
   Aj!@ AG
 @ ( "E
    6 J (! (!   6   6   (6 A )ðš7 Aj  Aj Atj(   (! (!   6   6 (!  A:     6 E
   6 J A j$ ´# Aà k"$ @@  (˜E
  A : L B 7@@@@@  ("- AG
  Aj œ (! (!@ - L"AÿF
  E
 A )ðš7 A0j AÀ j Aj Atj(   A : L Aj —@ (" ("F
  ! !@@ - L"AÿF
 @ AG
  ! !@ (@"E
   6D J (! (!  6D  6@  ( 6H A )ðš70 AØ j AÀ j A0j Atj(   (! (!  6D  6@ A: L  ( 6H  k! E
  6 J  6D  6@ A0j  (àº@@@@ - <AG
  (0 (4G
  Aj! - L!@  -  "AÿG
  AÿF
 AÿG
 A )ðš7 AØ j  Aj Atj(    Aÿ:    6  6  )7 Aj Aj   A0j½@@@ - ,
   Aj! - L!@  -  "AÿG
  AÿF
 AÿG
 A )ðš7X AÔ j  AØ j Atj(    Aÿ:    Aj A$j„@@ - ,E
  ((! A 6(  (!   6@ E
  ("E
	  Aj"6 
   ( (   - ,E
   Aj!  -  !@ (" ("G
  - L!@ AÿG
  AÿF
 AÿG
 A )ðš7X AÔ j  AØ j Atj(    Aÿ:  @ AÿF
 @ AG
 @ ( "E
    6 J (! (!   6   6   ( 6 A 6  B 7 A )ðš7X AÔ j  AØ j Atj(   (! (!   6   6   ( 6 A 6  B 7  A:  t  6T A )øš7X AÔ j  AÀ j AØ j Atj(    6T A )øš7X AÔ j  AÀ j AØ j Atj(   - ,AG
 Aj¼  6X A )øš7 AØ j  AÀ j Aj Atj(   - <AG
  (0"E
  !@  (4" F
 @  A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    Axj" ( !  A 6 @ E
  ^   G
  (0!  64  (8 kâC - L" AÿF
  A )ðš7 A0j AÀ j Aj  Atj(   Aà j$  *@  ("E
 @ ™
   ¢  A A £*@  ("E
 @ ™
   ¢   A £*@  ("E
 @ ™
   ¢   A£ @  (E
   ¢'@  (" E
     (Aj"6 
    '@  (" E
     (Aj"6 
    `# Ak"$ @@  -  AG
   (  (k!A !  (" E
   - AG
  Aj  œ (! Aj$  U@ -  AG
  (!   ("6     k6@ ("E
  - AG
    œ  B 7  @  (" 
 B   Õ€# Ak"$ @@ -  AG
  (!  ("6   k6@ ("E
  - AG
  Aj œ B 7  )7    Ž% Aj$ †@@@ -      (6    (6   (6 A 6 B 7¯  (! (!  A 6  B 7 @@ E
  AL
   AV"6     j6@@ Aq"
  !A ! !@  -  :   Aj! Aj! Aj" G
 @ AI
   j!@  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
    6Å6  Å6  @ ( "E
   6 J„~# Ak"$ @@  ( " - "AÿF
 @ 
   ) 7  A )ðš7 Aj   Aj Atj(    Aÿ:  ) !  A :    7  Aj$    (   ´ô# Ak"$ @@  - "AÿF
 @ AG
 @ ( " E
    6  J A 6 B 7   ( 6   (6  (6 A 6 B 7  A )ðš7 Aj   Aj Atj(    A 6  B 7   Aÿ:    ( 6    (6   (6 A 6 B 7   A:  Aj$ à# A0k"$ A !@@  E
  A6, A€›6(  )(7@   Ajï"
  A6$ AÃ¯$6   ) 7   Ajï"E
   ( ( 6 AjAÆ¯$‰! (!  A 6@  E
   ^ (" E
   Aj" 6  
   ( (   A0j$   ã
# AÀk"$ @@@@ ("
  (! ( !  A 6  B 7  E
 AL
   AV"6     j6@@ Aq"
  !A ! !@  -  :   Aj! Aj! Aj" G
 @ AI
   j!@  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
    6 ( !@@@@@ AG
  A F
 A!O
@@@@ E
  Aj Aj Ö6 A F
 Aj j" :   AO
 Aj Av:   AF
 Aj Av:   AO
 Aj :   AF
 Aj Av:   AF
A!  : “  Av: ”  :   Av: ’  Av: ‘ AF
A! AO
 Aj jAjAó‚±£6  A	!  j"A!O
  6Œ  Aj6ˆ  )ˆ7@ AÀ j A°j†% ( ! (AG
 A G
 (! A 6„  Aj6€  )€7   A j‡% AO
 (!  6|  A°j6x  )x7(  A(j‡% (!A !  A 6  B 7 @@ Apq"A j"
 A ! AL
   AV"6     j"6 A  Ø6   6  k" AM
   ApjK
   Aj"I
 Ø7:   AjØ7:   AjØ7:   AjØ7:   AjØ7:   AjØ7:   AjØ7:   AjØ7:   AjØ7:   A	jØ7:   A
jØ7:   AjØ7:   AjØ7:   A
jØ7:   AjØ7:   AjØ7:   ( ˆ% (!  6t  Aj6p ( !  6l  6h  )p7  )h7  Aj AjŠ%  j!   k! @ Aq"E
  Aj  j ×6 Aj jA k" Ø6 (!   6d  6` A6\  )`7  Aj6X  )X7   Aj Š% (! ( !	A !  A 6  B 7 @@ 
 A ! AL
   AV"6     j6@@ Aq"
  	! !A ! 	! !@  -  :   Aj! Aj! Aj" G
 @ AI
  	 j!@  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
    6  Aj"A AI6L  6P   k6T  A°j6H  )H70  )P78 A8j A0j% AÀj$ Å6 ©# Ak"$ @@@@  ( AAŒW"A 6 A:    Aj!@  ( " A G
  A 6Œ  6ˆ  )ˆ7 Aj Aj‡% !   A!O
@  E
  AØ j   Ö6 AØ j  j" :   AjAó‚±£6   Aj Av:   Aj :   Aj Av:   Aj Av:     A	j6D  AØ j6@  )@7  A j AÈ j†% A6<  AÈ j68  )87 Aj Aj‡% !   ( "A!O
@ E
  AØ j  Aj Ö6 AØ j j" :   Aj Av:   Aj :   Aj Av:   Aj Av:    Aj64  AØ j60  )07 Aj AÈ j†%  ( !AAˆW!   Aj"A AI6,  AÈ j6(  )(7    ÿ$ Aj$    œ~# Að k"$ @@ E
 @@@  (   ) "7  7h  AjÍ (!   ) "7(  7`  A(jÍ AÈ j Ç (L"  I
   k B ˆ§"I
  6\  (H  j6X  )X7   A j€% Aj! Aüj! (!  ( !A !	 ("
!@ 
 	I
A  k"   I" 
 	kK
  AO
@ E
    j  	j ×6 (!     j" 6  k!  	j!	@  AG
 @ -  AG
   ˆ%A !  A 6 A :  A!  	 
O
  A6D A6<  68  AÈ j6@  )@7  )87  Aj Aj‰% A64  AÈ j60  )07   ÍA !  A 6 
  Að j$  A G ¿# AÀ k"$ @ E
 @@  (  (AG
  A6, A6$  Aüj6   A0j6(  )(7  ) 7 Aj Aj Aj‰% - ?" AK
  A  kAÿq6  A0j6  )7   Í J AÀ j$  A GÔ# AÐ k"$  A4jÃ!    ·!@@ ( "
 A !A ! Aj! (!  60  6,  ),7    Aj ¸    ¹ Aj È B 7 @ (E
   )7   ) 7 AÌ j Ajý!  Ä  ( ! AÐ j$  Î# A°k"$ @ E
   (Aj"6@ E
  (! (! Aj! !A !A !	A !
A !A !@ Aˆj Ö!
@@@@@ 
×"E
 @@ (Œ"
 A ! à!@ A…›‰E
  A6„ A€›6€  )€7(@  A(jˆ
  A6| AÃ¯$6x  )x7   A jˆE
A  
 k"AtAj 
 F" G
@  	 k"O
 A€ ÝC!@ 	 
G
 @@  F
  A|j 6  ! !A 	 k"Au "A€€€€O
 At"	ÝC" AjA|qj"!
@ 
   j!
 !@  ( 6  Aj! Aj" 
G
   	j!	@ E
   âC A|j 6  ! ! 
 	G
 
!	@  M
  	 k!   kAuAjA~mAtj!@ 	 F
    ×6  j!
 !A 	 k"Au 	 F"A€€€€O
 At"ÝC" A|qj"!
@ 	 F
   	 kj!
 !@  ( 6  Aj! Aj" 
G
   j!	 E
   âC 
 6  
Aj!
A Au 	 F"A€€€€O
 At"ÝC" j!	  j!A€ ÝC!@  G
 @  M
   AuAjA~mAtj!A Au "A€€€€O
 At"ÝC!  âC  j!	  A|qj! !  6  Aj! 
 F
@@@  F
  !@  	O
   	 kAuAjAmAtj"  k"k!@  G
  !   ×6 !A 	 k"Au 	 F"A€€€€O
 At"	ÝC" AjA|qj"!@  F
    kj! !@  ( 6  Aj! Aj" G
   	j!	  âC ! ! A|j" 
A|j"
( 6  
 F
  @ õE
    ( (D "È"6t      Aô jº6P  AÐ jÉ (P! A 6P@ E
  ^ E
  ^@ óE
 @  ( (@ "E
   (Aj"6 E
A$ÝC Ÿ" (Aj"6 E
 §@@  (AG
  ªAK
  B 7 B 7h  Ajˆ AÐ jÃ" ª"Apj   (AFË    ·! AÈ j «  )H7    Aj ¸!    ¹!@@ E
  E
  A<j Ê  A<j• (<"E
  6@ J B 7 B 70  Ajˆ Ä ("E
  Aj"6 
   ( (  @ E
  ("E
  Aj"6 
   ( (   ("E
  Aj"6 
  ( (  A !@@ !@ "
 A ! !@  Aj"AvAüÿÿqj"(  AÿqAt"j"( "E
   (Aj"6 E
@ ("E
   (Aj"6 E
 (  j"(! A 6@ E
  ("E
  Aj"6 
   ( (   ( ! A 6 @ E
  ("E
  Aj"6 
   ( (  @A  
 kAtAj 
 F kAxjAÿwK
  
A|j"
( A€ âC@@ µ"E
  !@  G
  !@ E
   (Aj"6 E
	 ! E
  ("E
  Aj"6 ! 
   ( (   !@ E
  ("E
  Aj"6 
   ( (  @ E
  ("E
  Aj"6 
   ( (   
  
Ñ 
 !@ 
 F
  
! ( "  AvAüÿÿqj(  AÿqAtj"F
  !@ (! A 6@ E
  ("E
  Aj"6 
   ( (   ( ! A 6 @ E
  ("E
  Aj"6 
   ( (  @ Aj" ( kA€ G
  Aj"( !  G
  
!@  kA	I
 @ ( A€ âC 
 Aj"kAK
  
!@  F
 @ ( A€ âC Aj" G
 @ E
   	 kâC ("E
  Aj"6 
  ( (  ð@ E
   âC ! ! !
  AvAüÿÿqj(  AÿqAtj" 6  6  
Õ Aj!     A°j$  A G¢ (!  B 7   6  AjB 7   AjB 7   A jB 7   A(jA 6    A  A I"6 @ E
 @ E
   Aj (  ×6 AG
 AAôW!  Aj"( !  6  E
  J     (!  A 6@ E
  J      6   6   c~# Ak"$ @@ (
   A 6  B 7  (! ( !  ) "7   7    A  ¶ Aj$    A 6   Ú# A k"$   A 6  B 7  A˜›6  B 7  Aj!@ (E
   ) 7  )7  Ajý!  A: @ ( "E
  ("E
  Aj   @ (("E
   ( Aj6   6  Aj„ (! A 6 E
  ^ A j$   Å# Ak"$   A 6  B 7  A˜›6    ( "6@ E
   ( Aj6   A : @ ( "E
  ("E
  Aj   Aj" @ (("E
   ( Aj6   6  Aj„ (! A 6 E
  ^ Aj$   …~# A k"$   ( "6@ E
   ( Aj6   ) "7  7  AjÀ"6   Aj AjÂ! @ E
  ^@ E
  ë A j$   #  (!  A 6@ E
  ^  Ô(  (!  A 6@ E
  ^  ÔAâC AAAÝC"B7 A :  B 7 A˜›6  Aj  Ajƒ   - :   @  (" E
     ( Aj6   
   Aj ƒ   ^# Ak"$ @@  (" 
 A !A !   Aj!  (!    6  6  )7  ¿!  Aj$   ß# Aà k"$ A !@@  ("
 A !A ! Aj! (!@ E
   6P  6L  )L7( AÔ j  A(j¿ (X (T"k! ! A  !@@  - AG
   6H  6D  )D7 AjÂ!  6@  6<  )<7  A jÁ!@@ E
  (! B 74@ E
   68  Aj64  )47  Aj°! ^ B 7 B 74  Aj°!@ E
  J Aà j$  Ç# A k"$ @@  - AG
 @@  (" 
  B 7  (! B 7 E
   6   Aj6  )7  Â! @@  (" 
  B 7  (! B 7 E
   6   Aj6  )7 AjÁ!  A j$   ¡~# A k"$ B !@@@ ( ! A6  Aj6 ( (!  )7  Aj   E
@ ( A¥ ‘²G
    7 A! B|"BR
 A !  A :     :  A j$ æ# Ak"$   (!A ! A 6 Aj Aj j@@ E
   ( !A ! A !@@@   j-  "A#G
   Aj" O
   Aj" O
A !A ! @  j,  "A€q
 A !  “7E
 A9A  A`j  AŸjAI" ÀA9J  jAt! @  j,  "A€q
  “7E
 AIAP A`j  AŸjAI"ÀA9J j!  (O
   j! !   (O
 ( j :   Aj!  Aj"  I
  Aj k (! Aj$   ü
# Ak"$ @@@  ( "
 A !@@ ("E
  Aj! Aq!@@ AG
 A ! !  A~q!A ! ! A !@A!	@  -  À"
A H
 A!	 
A#F
 AAA 
A€œj-  "	AÄ F 	A× F!	  	j!A!	@  Aj-  À"
A H
  
A#F
 AAA 
A€œj-  "	AÄ F 	A× F!	  Aj!   	j! Aj" G
 @ E
 A!	@  -  À" A H
   A#F
 AAA  A€œj-  " AÄ F  A× F!	  	j!  G
  ( Aj6   j! A 6 Aj Aj jA ! @@@@ -  "	ÀA H
  	Aÿq"
A#F
  
A€œj-  A¼j     (O
 (  jA#:   ("  Aj"
I
  
kAM
 	Aÿq ( 
j«  Aj!    (O
 (  j 	:    Aj!  Aj" G
  Aj  k (! Aj$   ‡  A 6  B 7 @@ E
  A€€€€O
   At"ÝC"6     j"6A ! A  Ø6!   6@  Atj  Ÿ8  Aj" G
 Å6 ­~~~# Ak"$ @@  
 A ! A6 AìÏ6  )7    ô!  ) !@@  
 B !  ("­B †  AjA  ­„!A !@ B ˆ B ˆ"R
  § § §°7E!  E
   ^ Aj$  í	~~# A k"$ @@  
 A ! @@@ Aj  •"( " (E
 @  ("  Aj"F
  ) "B ˆ"§! §!	@ (×"E
@ é" E
     (Aj"
6 
E
 ("
E
  
Aj"
6@ 

   ( (    E
 A6 AìÏ6  )7@@   Ajô"
 B ! ("
­B † AjA  
­„!A !
@ B ˆ R
  § 	 °7E!
@ E
  ^  ("E
   Aj"6@@ 
     ( (   

A !  

 A ! @@ ("E
 @ " ( "
  @  (" ( G!  ! 
   !   G
 A!  A !  ˜ A j$   9# Ak"$  A6 A¿Š6  )7    Ó!  Aj$   Þ~~~# A k"$ @@  
 A !  A6 AìÏ6  )7@   Ajˆ
 A!  A6 AìÏ6  )7    ô! ) !@@ 
 B ! (" ­B † AjA   ­„!A ! @ B ˆ B ˆ"R
  § § §°7E!  E
  ^ A j$   —~# Ak"$ @@@ 
   Aî´Aˆ@@@@@@@@  ( ( Aj	   Aî´Aˆ  Aø£Aˆ!   ( ( 6  Aj“ (! A 6 E
 ^  õÍ6   Aj“ (! A 6 E
 ^   ( ( "6  Aƒ–Aˆ!  AjÐ6  Aj“ (! A 6@ E
  ^ E
 ^  Aø£Aˆ ñ(š9A¶£Aˆ å!  AóAˆ@ ( (F
 A !@@@  ¥"(E
   Aø£Aˆ (š9AþAˆ   Ö ("E
  Aj"6@ 
   ( (   Aj" ( (kAuI
   AíòAˆ Aj é•!  Aý‡Aˆ ( "(E
@ (" Aj"F
 @  Aƒ–Aˆ!  AjÐ6  Aj“ (! A 6@ E
  ^@@ ("(E
   Aø£Aˆ ((š9A¶£Aˆ   Ö@@ ("E
 @ "( "
  @  ("( G! ! 
  !  G
   Aº‡Aˆ ˜@ ó"E
   (Aj"6 E
   à"Ö"A¶™Aˆ@ E
  ("E
  Aj"6 
   ( (  A$ÝC Ÿ" (Aj"6 E
 §  «  ) "§ B ˆ§ 9 AŸ´Aˆ ("E
  Aj"6 
   ( (   Aj$    ¾# Ak"$   A 6  B 7  Aˆž6    ( "6@ E
   ( Aj6 @ ( "E
  ("E
  Aj   Aj" @ (("E
   ( Aj6   6  Aj„ (! A 6 E
  ^ Aj$   #  (!  A 6@ E
  ^  Ô(  (!  A 6@ E
  ^  ÔAâC AL# Ak"$ AÝC! A 6  Aj  Aj×"   (Aj"6@ 
   Aj$    @  (" E
     ( Aj6   
   Aj ƒ   ^# Ak"$ @@  (" 
 A !A !   Aj!  (!    6  6  )7  ¿!  Aj$   Ü# A0k"$  A6, Aƒ–6(  )(7@@  Aj°
 A !@  (" E
     ( Aj6    6$ A$jÐ!  ($! A 6$@ E
  ^@  
 A!@@  ("
 A!  6    Aj6  )7  Aj°!  ^ A0j$  3   A 6  B 7  B 7  B 7  Aìž6     Aj6  M   B 7  B 7  Aìž6    ( "6@ E
   ( Aj6   B 7    Aj6  ¸  A6  Aj!@  ("  Aj"F
 @@ ((AG
  A 6@@ ("E
 @ "( "
  @  ("( G! ! 
  !  G
    (ä  (!  A 6@ E
  ë  Ôƒ@@ E
    ( ä   (ä (!  A 6@  E
   ("E
   Aj"6 
     ( (   (!  A 6@  E
   ^ AâC ½  A6  Aj!@  ("  Aj"F
 @@ ((AG
  A 6@@ ("E
 @ "( "
  @  ("( G! ! 
  !  G
    (ä  (!  A 6@ E
  ë  ÔA$âC A   	   A Ù³# A k"$  Aj"! !@@ ("E
 @@   "("O
  ! ( "
   O
 ("
  Aj!AÝC" 6 B 7    6  6 @ ( ( "E
   6  ( ! ( È  (Aj6A$ÝC"A 6 Aìž6  B 7   ("6@ E
   ( Aj6  B 7 A6  Aj6    (Aj"6@ E
     ("Aj"6 E
 @@  ("	  Aj"
G
    6 Aj! Aj!@@@ ( "E
  	(!
 !@   ( 
I"!  Atj( "
   F
  
 (O
  6 B 7 Aj (  ê@ 	("  Aj ( (L "E
   	("6@ E
   ( Aj6   6 Aj  Aj Ajë (! A 6@ E
  ("E
  Aj"6 
   ( (   (! A 6 E
  ^ Aj (ì@@ 	("E
 @ "( "
  @ 	 	("( G! !	 
  !	  
G
     (Aj6  ("E
   Aj"6@ 
     ( (   A j$   “@  F
   Aj!@  (! !@@@   ( F
  ! !@@ E
 @ "("
  @  ("( F! ! 
  ( ("I
  ! ! E
@@  "("O
  ! ( "
  O
 ("
  Aj! Aj  "( 
   !AÝC! (!  6 B 7   6  6 @  ( ( "E
    6  ( !  ( È    (Aj6@@ ("E
 @ "( "
  @  ("( G! ! 
  !  G
 Š Aj!@@@ ("
  !@@  "Aj"E
  ! ( "
@  E
  Aj! ("
A ! ( "
AÝC! ( ! A 6   6 (! A 6  6 B 7   6  6  !@ ( ( "E
   6  ( ! ( ÈA!  (Aj6   :    6 % @ E
    ( ì   (ì AâCÅ~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    (Aj"6  ! 
 A ! A j$  Å~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    (Aj"6  ! 
 A ! A j$  ×~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    ( (T " E
    (Aj"6  ! 
 A ! A j$  ×~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    ( (T " E
    (Aj"6  ! 
 A ! A j$  Æ~# A k"$ @@  ("
 A ! ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
 @   G
 A !  7  7A !  Aj AjŒA J
   (" E
     ( ( ! A j$  Ø~# A0k"$ @@  ("E
  ) !  Aj"! @  7  7(    Aj Aj‹"!  AA  j( "
    F
   7  7(  Aj AjŒA J
   ("E
   ( ( !  ) "7   7 A$j ý( ! A0j$  Ý~# A k"$ @@  ("
 A ! ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
 @   G
 A !  7  7A !  Aj AjŒA J
   (" E
 @  ñ"E
  Ñ" E
    ( ( ! A j$  È~# A k"$ @@  ("
 A ! ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
 @   G
 A !  7  7A !  Aj AjŒA J
   (" E
   ë" E
   Ü! A j$  ¼~# A k"$ @  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7  Aj AjŒA J
   ("E
  çE
   ( ( A G! A j$  Æ~# A k"$ @@  ("
 A ! ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
 @   G
 A !  7  7A !  Aj AjŒA J
   (" E
     ( ( ! A j$  ±~# A k"$ @  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7  Aj AjŒA J
   ("E
   ( ( ! A j$  È~# A k"$ @@  ("
 A ! ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
 @   G
 A !  7  7A !  Aj AjŒA J
   (" E
   ï" E
   ù! A j$  º}~# A k"$ C    !@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7  Aj AjŒA J
   ("E
   ( ( ! A j$     é~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    ( (T " E
    ( (X " E
    (Aj"6  ! 
 A ! A j$  é~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    ( (T " E
    ( (X " E
    (Aj"6  ! 
 A ! A j$  š~# A0k"$ @@@  ("E
  ) !  Aj"!@  7  7(   Aj Aj‹"! AA  j( "
   F
   7  7( Aj AjŒA J
  ("E
   ( (T "E
   ( (X "E
   (Aj"6 
  ) "7  7  A(j Ajý!A$ÝC"A 6 Aìž6  B 7   ("6@ E
   ( Aj6  B 7 A6  Aj6@    þ"E
   (Aj"6 E
 ( ! A 6  E
  ^ A0j$   ‘# A k"$ @  (
 @@ 
 @  ("
 A !  Aj!  Aj"! @    Aj "!  AA  j( "
 A !   F
   Aj
   Œ (
 ó
  Aj!@@@  (" E
   (" E
  Aj     ((" 
A !  ( " 
 A !     ( Aj6    6  Aj6 Aj  AjAý$ Aj AjŽ ("(!   6@  E
   ("E
   Aj"6 
     ( (   (!  A 6  E
   ^ A j$   á~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    ( (T " E
  å" E
    (Aj"6  ! 
 A ! A j$  á~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    ( (T " E
  å" E
    (Aj"6  ! 
 A ! A j$  ¿~# A0k"$   ) "7  7(@@   Aj€"
   7  7 A$j Ajý!A$ÝC  Aj–" (Aj"6 E
@    þ"E
   (Aj" 6  E
 ( !  A 6   E
   ^ A0j$   á~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    ( (T " E
  ó" E
    (Aj"6  ! 
 A ! A j$  á~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
    ( (T " E
  ó" E
    (Aj"6  ! 
 A ! A j$  Ï~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
  ï" E
    (Aj"6  ! 
 A ! A j$  Ï~# A k"$ @@  ("E
  ) !  Aj"! @  7  7    Aj Aj‹"!  AA  j( "
    F
   7  7A !  Aj AjŒA J
  (" E
  õ" E
    (Aj"6  ! 
 A ! A j$  Ù~# A k"$ @@ ("E
  ) ! Aj"!@  7  7   Aj Aj‹"! AA  j( "
   F
   7  7 Aj AjŒA J
  ("E
   ( (T "E
  å"E
    ž  B 7   AjB 7  A j$ é~# A k"$ @@ ("E
  ) ! Aj"!@  7  7   Aj Aj‹"! AA  j( "
   F
   7  7 Aj AjŒA J
  ("E
   ( (T "E
  å"E
       B 7  B€€€€€€€À?7  B€€€ü7  A j$ ©~# A k"$   Aj!@@  (" E
  !@  ) "7  7     Aj Aj‹"!  AA  j( " 
   F
   ) "7  7 Aj AjŒAH
 ! A j$   GÂ  A 6  B 7   (Aj"6@@ E
   ("Aj"6 E
 @@ (" Aj"F
 A !@ Aj!@@   (O
   ( "6 @ E
   ( Aj6  Aj!   Š!   6@@ ("E
 @ "( "
  @  ("( G! ! 
  !  F
    6  (Aj6 ("
   Aj"6@ 
   ( (  û@@  ("  ( "kAu"Aj"A€€€€O
 @@  ( k"Au"   KAÿÿÿÿ AüÿÿÿI"
 A ! A€€€€O
 AtÝC!  Atj" ( "6  At!@ E
   ( Aj6   ( !  (!  j! Aj!@  F
 @ A|j"( ! A 6  A|j" 6   G
   (!  ( !   6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ^  G
 @ E
    kâC Å6 ð      þÖ@@ ("
  !@  ("( G! ! 
  @ "( "
 @  (  G
    6     (Aj6  ( ï (! A 6@@ E
  ("E
  Aj"6 
   ( (   (! A 6@ E
  ^ AâC  û~~}}# Ak"$ @@ ( "
  B 7 (! B 7 E
   6  Aj6  )7  ”!@@@ ("
 @@ i"AK
  Aj q!	 !	  I
   p!	 (  	Atj( "E
  ( "E
  Aj!
 ( "Aj! AK!
@@@ (" F
 @@ 

   
q!  I
   p!  	G
B !@@ ("
 A !B !A  Aj 5"P!@ E
  ("­B † A  ­„!  B ˆR
   § §°7
 A ! ( "
 AÝC" 6 A 6   ( "6@ E
   ( Aj6  *! (Aj³!@@ E
   ³” ]E
 At AI  AjqA Grr!@@  •"C  €O] C    `qE
  ©!	A !	A!@  	  	K"AF
 @  Ajq
  ! Æ8!@@  ("K
   O
 AI!@@ (³ *•"C  €O] C    `qE
  ©!	A !	@@ 
  iAK
  	AA  	Ajgkt 	AI!	 	Æ8!	  	  	K" O
  ™@ (" Aj"q
   q!	@  O
  !	  p!	@@ (  	Atj"( "
   Aj"( 6   6   6  ( "E
 (!@@  Aj"q
   q!  I
   p! (  Atj 6   ( 6   6 A!  (Aj6   :    6  Aj$  Aj!@@@ ("
  !@@  "Aj"E
  ! ( "
@  E
  Aj! ("
A ! ( "
AÝC! ( ! A 6 ( ! A 6  (!  6@ E
  ^  6 B 7  A 6  6  !@ ( ( "E
   6  ( ! ( ÈA!  (Aj6   :    6 ç@  (
 @  ("E
   Aj"! @    Aj "!  AA  j( "
    F
    Aj
   (ñ
 @  ("E
   (Aj"6 E
  ’  ("  ( (P !  (!   6 E
  (" E
   Aj" 6  
   ( (   Ý~# A k"$ @  (
 @@  ("
 A !  Aj!  Aj"! @  ) "7  7    Aj Aj‹"!  AA  j( "
 @   G
 A !  ) "7  7A !  Aj AjŒA J
   (!  A 6   Œ A j$   Ö# A k"$ @  (
 @  ("E
   Aj!  Aj"!@   Aj "! AA  j( "
   F
   Aj
 @@ ( "E
  !@   Aj "! AA  j( "
   F
   AjE
 !  F
 @@@  ("E
  ("E
  Aj    (("
A ! ( "
 A !  ( Aj6   6  Aj6 Aj  AjAý$ Aj AjŽ (! (! A 6 (!  6@ E
  ("E
  Aj"6 
   ( (   (! A 6@ E
  ^  Œ A j$  ¯A$ÝC  Aj–" (Aj"6@ E
 @    þ" E
     (Aj"6 E
AÝC * ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (  AÝC *ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (  AÝC *ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (  AÝC *ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (    ("E
    Aj"6@ 
     ( (   ÉA$ÝC  Aj–" (Aj"6@ E
 @    þ" E
     (Aj"6 E
AÝC * ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (  AÝC *ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (  AÝC *ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (  AÝC *ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (  AÝC *ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (  AÝC *ò" (Aj"6 E
 @   ¼"E
 @ (Aj   ( (    ("E
    Aj"6@ 
     ( (   # AÐ k"$  A6L Aý‡6H  )H7(A !@@  A(j°E
   µ!    (Aj"6 E
    (Aj"6 E
@@  ("  Aj"F
 @ A6D Aƒ–6@  )@7 A !  A j°E
@@ Aj"Ð"	E
  	(!
 B 78@ 
E
   
6<  	Aj68  )87  Aj°!
 	^ 

 B 7 B 78  Aj°E
 (!	@@ E
 A !
 AÍ“‰
 !
 	  
 	( (H E
 !
@@ ("	E
 @ 	"( "	
  @ 
 
("( G!	 !
 	
   G
  A64 Aº‡60  )07  Aj°!    (Aj6  ("E
   Aj"6 
     ( (   AÐ j$   8   6   (Aj"6@ E
   ( " (Aj6       6   (Aj6      6   (Aj6  \  ( " (Aj6  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     É@@@ E
  A€€€€O
 AtÝC!  ( !   6 @ E
    (AtâC   6 Aq!A !A !@ AI
  Aüÿÿÿq!A !A !@  (  At"jA 6   (  jAjA 6   (  jAjA 6   (  jAjA 6  Aj! Aj" G
 @ E
 @  (  AtjA 6  Aj! Aj" G
   ("E
  Aj! (!@@ i"AK
   Ajq!  I
   p!  (  Atj 6  ( "E
 Aj! AK!@ (!@@ 
   q!  I
   p!@@  G
  !@  (  At"j"	( 
  	 6  ! !  ( 6    (  j( ( 6   (  j(  6  ( "
    ( !  A 6 @ E
    (AtâC  A 6ð     6   J  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     Æ# A0k"$ @@  ( "
 A !  A6( Aêó6$A!  A6  A±‡6  )$7  )7   Aj Ajò6,@ A,jA †‰
 A!  A,jAþú‰
 AA  A,jAâþ‰!  (,! A 6, E
  ^ A0j$   P# Ak"$ @@  ( " 
 A !  A6 Aœ†6  )7    A õ!  Aj$   Ò}}}}}}# A0k"$  * *“! * * “! *! * !C  €?!	C  €?!
@@@@ œ   C  €?  C  €?]•!	 C  €?  C  €?]•!
C  €?!	C  €?!
@  ]E
  C  €?  C  €?]•!
  ]E
 C  €?  C  €?]•!	C  €?!	C  €?!
@  ^E
  C  €?  C  €?]•!
  ^E
  C  €?  C  €?]•!	@@ ( "E
  A6( Aþú6$ A6  A¼þ6  )$7  )7   Aj Ajò6, A,jA±‡‰! (,! A 6,@ E
  ^ 
 	 
 	 
]"	!
   	8   
8  A0j$ ­	}}}}}}# Ak"$ C    !@@@ ( "
 C    ! A6 A±‡6  )7 C    !  ÿ"E
 C    !@@ (" ("	G
 C    ! A Ÿ!  	kAI
  AŸ! ("E
  Aj"6 
   ( (   * !
 *! * ! * !
    * *“ * *”“”8     
“ 
 ”“”8  Aj$      6   J  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     I~# Ak"$ @@  ( " 
 A !   ) "7   7   ˆ!  Aj$   N# Ak"$ @@  ( " 
 A !  A6 A þ6  )7    ö!  Aj$   Î~}}}}# Ak"$ @@ ( "
   B 7   ) "7   7@@  ÿ"E
 A !A !@@@@ ( (kAuAj @@ A ŸC  C”"‹C   O]E
  ¨!A€€€€x! At Atr rA€€€xr!A!@@ A ŸC  C”"‹C   O]E
  ¨!A€€€€x! At!@@ AŸC  C”"‹C   O]E
  ¨!A€€€€x!  Atr!A!@@ AŸC  C”"‹C   O]E
  ¨!A€€€€x!  rA€€€xr!A! A Ÿ! AŸ!	 AŸ!
@@C  €? 	 AŸ"’"	C  €? 	C  €?]“C  C”"	‹C   O]E
  	¨!A€€€€x! At!@@C  €? 
 ’"	C  €? 	C  €?]“C  C”"	‹C   O]E
  	¨!A€€€€x!  r!@@C  €?  ’"C  €? C  €?]“C  C”"‹C   O]E
  ¨!A€€€€x!  AtrA€€€xr!   6   6  (" E
   Aj" 6  
  ( (    B 7   Aj$ ¡~}}}# Ak"$ @@@ ( "
   B 7   AjA 6   AjB 7   ) "7   7@@@  ÿ"E
 @@@ ( (kAuAj  A Ÿ! AŸ! AŸ!  A 6   8   8   8  A6  A Ÿ! AŸ! AŸ!   AŸ8   8   8   8  A6   B 7   AjA 6   AjB 7   B 7   AjA 6   AjB 7  A Ÿ!  A 6  B 7   8  A6  (" E
   Aj" 6  
   ( (   Aj$  I~# Ak"$ @@  ( " 
 A !   ) "7   7   ó!  Aj$   I~# Ak"$ @@  ( " 
 A !   ) "7   7   ƒ!  Aj$   T# Ak"$ @@ ( "
 A ! A6 A¤„6  )7   û!   š Aj$ P# Ak"$ @@  ( " 
 A !  A6 A®þ6  )7    A ÷!  Aj$   ~ ) !  A 6   7      ¯@@ (" ("O
  ( !@  Aj"6@A€œ  j-  "j-  "A× G
 @  O
  Aj"6  j! !A€œ -  "j-  "A× F
  A%G
    K!@  F
  Aj"6  j! !@ -  Avj    I
   B 7 @  Aj"M
 @ AÄ F
    ­@@@@@@ AXj    ®   ¯@  O
   j-  A>G
   Aj"6  k"  kK
   6    j6 @@  I
  !A!@  Aj"6@@@  j-  AXj  Aj! Aj!  O
 ! A J
   k"  kK
  B 7   F
   6    j6   A6    j6  ¾ ("!@  ("O
  ( ! !@@A€œ  j-  j-  A¼j                     Aj"6  G
  !@  Aj"I
   k"  kK
  ( !  B 7 @  F
    6    j6  ¸@@@ (" ("O
  Aj! ( !@@@A€œ  j-  j-  A¼j    I
  k"  kK
  B 7   F
   6    j6   Aj"6  G
   B 7  ž ("Aj!@@  ("I
   M
 ( !  A6    j6   Aj"6@@@ ( " j-  "A<F
 @  I
  ! A>G
 !  I
  kAM
  A6    j6 @  Aj"6  O
  j! ! -  AÿqA>G
   I
   k"  kK
   B 7 @  F
    6    j6  %    ( "6 @ E
   ( Aj6   Ñ# A0k"$  A6( A­‡6$ A 6,  )$7A!@  Aj©"E
    ( ( 6 A,j Aj„ (! A 6@ E
  ^ (,"E
  (E!@ E
  E
  A6 A­‡6  )7   Ajñ6  A,j A j„ ( ! A 6  E
  ^@@ E
  ("E
  Aj"6 
   ( (     (,"6 @ E
   ( Aj6  ^ A0j$       ( !  A 6 @ E
  ^  ‡}# Aà k"$ @@@ ( "E
  ("
  A :   A :    6P  Aj6L  )L7 AÔ j Ajª! A6H A€È6D  )D7@@  AjA´
   A:   B 7  B 7< A(j ¬@@ (("
  B 70@ (,"AK
  B 70  Aj64  Aj60  )07  AjÏ68 A<j A8j„ (8! A 68@ E
  ^ A j ¬  ) 7   ò"8@ (<!  A:    8   6  « Aà j$ ©~~# Ak"$ @ Aj"A€€€€O
 A ! At"ÝCA  Ø6!  A 6 A 6  Aj  ¬@ (E
  A G! ) "	B ˆ"
§! 	§!A!
A!@@ )"	B ˆ 
R
  	§  °7
   H
     Atj( 6A!  Atj  (6  Aj  ¬A !A  Aj  F! 
Aj"   H!
 (
   âC Aj$  Å6 K}# Ak"$  Aj  ³C    !@ - AG
  *! (" E
   ^ Aj$  }}}# AÐk"$ @@@ ( "E
  ("
  A :   A :    6À  Aj6¼  )¼7XA! AÄj AØ jª! A6¸ AÓÆ6´  )´7P@@  AÐ jA´E
  A¬j ¬  )¬7  ò!  A 6  B 7   8  A6  A6¨ AÉÄ6¤  )¤7H@  AÈ jA´E
  Aœj ¬  )œ7 Ajò! A”j ¬  )”7 Ajò! AŒj ¬  )Œ7 Ajò!  A 6   8   8   8  A6 A!A! A6ˆ AÏ¿6„  )„7@@  AÀ jA´E
  Aü j ¬  )|78 A8jò! Aô j ¬  )t70 A0jò! Aì j ¬  )l7( A(jò! Aä j ¬  )d7    A jò8   8   8   8  A6 A !  A :     :  « AÐj$ Š}}# A k"$  Aj ¶@@@ - 
 A !  A :  @@@@ (Aj @@ *C  C”C   ?’"‹C   O]E
  ¨!A€€€€x!   At Atr rA€€€xr­B †B„7 @@ *C  C”C   ?’"‹C   O]E
  ¨!A€€€€x! At!@@ *C  C”C   ?’"‹C   O]E
  ¨!A€€€€x! At r!@@ *C  C”C   ?’"‹C   O]E
  ¨!A€€€€x!    rA€€€xr­B †B„7 @@C  €? *" *’"C  €? C  €?]“C  C”C   ?’"‹C   O]E
  ¨!A€€€€x! At!@@C  €? * ’"C  €? C  €?]“C  C”C   ?’"‹C   O]E
  ¨!A€€€€x! At r!@@C  €?  *’"C  €? C  €?]“C  C”C   ?’"‹C   O]E
  ¨!A€€€€x!    rA€€€xr­B †B„7 A!   :  A j$      6   6   6   Q  (!  B 7@@ E
  ("E
  Aj"6 
   ( (    A 6    ># Ak"$  (! A6 A¢‘6  )7     † Aj$ ¨# A0k"$   (!  A6, A»þ6(  )(7@@@   Ajû"
 A ! A6$ Aâþ6   ) 7@@  Ajû"
 A !  (Aj" 6  E
 Aj —"( "(E
A !@ ("  Aj"F
 @@  Aj"AÔÇ‰
  ( " E
    ( Aj6   !  !@@  ("E
 @ " ( "
  @  (" ( G!  ! 
    G
  ˜ (" E
   Aj" 6  
   ( (   (" E
   Aj" 6  
   ( (   A0j$   Ã# A k"$    »"6  ( ! A6 AÀ‰6  )7@@  Aj·"E
 @ å"E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   E
  ("E
  Aj"6@ 
   ( (     (   Åù6 Aj Aj„ (! A 6@ E
  ^ (!@@ E
  (
 AjAª (! A j$   ­# A0k"$    »"6,  ( ! A6( AÀ‰6$  )$7A !@@  Aj·"E
 @ å"E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   E
     (   Å§6  A,j A j„ ( ! A 6 @ E
  ^ (,! !@@@ E
  (
 A,jAª (,"
 A ! A ! Aj!  (!  6   6  )7 Aj¿! @ E
  ("E
  Aj"6 
   ( (   (,! A 6,@ E
  ^ A0j$    Ñ~~# Ak"$   »!  (!  A6 Añú6  )7 B !@@   ñ" 
 A !B !A   Aj  5"P!@ E
  ("­B † AjA  ­„!A !@  B ˆR
   § §°7E!@  E
   ^@ E
  ^ Aj$  ž~~# Ak"$   ( ! A6 AÁô6  )7 @@@  ·"
 A !B !  ( ( !  »! @@ 
 A !B !A  Aj 5"P!@  E
   ("­B †  AjA  ­„!A !@  B ˆR
   § §°7E!@  E
   ^@ E
  ^ ("E
  Aj"6 
   ( (   Aj$   ¨~~# A0k"$   (! A6, Añú6( A6$ AÔÇ6   )(7  ) 7  Aj Ajò! AjAÔÇü!@ E
    »6  Aj„ (! A 6 E
  ^B !@@ 
 A !B !A  Aj 5"P!@ ( "E
  ("­B † AjA  ­„!@@@  B ˆR
   § §°7E
@  ( AjAñúü" ô" E
   ("E
   Aj"6 
     ( (   ( !  A 6 @  E
   ^ ( ! A 6 @ E
  ^@ E
  ^ A0j$  ‡# A k"$   (! A! A6 A´ƒ6 A6 A±€6  )7  )7 @   Aj ò" E
 @@  (AG
   - AÎ G
 A !@  (AG
   - AÉ G
 A!@  (AG
   - AÏ G
 A!@  (AG
   - AÐ G
 A!A!  (AG
   - AÔ G
 A!  ^ A j$  r~# A0k"$   (!  A6, AÚÿ6(  )(7 A$j   Ajü !   ) "7  7   Aj¢!  ¡ A0j$  X# A k"$   (!  A6 AÚÿ6  )7 Aj   Ajü " £!  ¡ A j$  p~# A0k"$  (! A6, AÚÿ6(  )(7 A$j  Ajü !  ) "7  7    Aj¤ ¡ A0j$ p~# A0k"$  (! A6, AÚÿ6(  )(7 A$j  Ajü !  ) "7  7    Aj¥ ¡ A0j$ r~# A0k"$   (!  A6, AÚÿ6(  )(7 A$j   Ajü !   ) "7  7   Aj¦!  ¡ A0j$  r~# A0k"$   (!  A6, AÚÿ6(  )(7 A$j   Ajü !   ) "7  7   Aj§!  ¡ A0j$  T# A k"$  (! A6 AÚÿ6  )7   Aj  Ajü "¨ ¡ A j$ X# A k"$   (!  A6 AÚÿ6  )7 Aj   Ajü " ©!  ¡ A j$  Ã# AÀ k"$  (! A6< AÈŸ68  )87@@  AjˆE
  (! A60 AÈŸ6,  ),7   Ajñ64   A4j° (4! A 64 E
 ^ ( ! A6( AÈŸ6$  )$7@@  Aj·"E
    ( ( 64   A4j° (4! A 64@ E
  ^ ("E
  Aj"6 
  ( (     (ä  AÀ j$ õ# A0k"$   (! A6, AËŸ6(  )(7@@@  AjˆE
   (!  A6$ AËŸ6   ) 7    A ÷!   ( ! A6 AËŸ6  )7@  Aj·"
   (å!   ( ( !  ("E
  Aj"6 
   ( (   A0j$    J  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     ‚~# A k"$ @@@  ((" 
 A !    (Aj"6 E
 A6 Až›6  )7@@   Ajü"
 A !  ) "7   7@@  ü"
 A !AÝC" 6  ("E
  Aj"6 
   ( (    ("E
   Aj"6 
     ( (   A j$   þ~# AÀ k"$ @@@  (("
 A !   (Aj"6 E
 A6< Až›68  )87@  Ajü"
 A$ÝC  Ajâ"("AF
  Aj"6 E
   ’ A4jAž›ü! (!AÝC   Å" (Aj"6 E
@   þ"E
 @ (Aj   ( (   ( ! A 6  E
  ^  ) "7  7(@  Ajü"
 A$ÝC  Ajâ"("AF
  Aj"6 E
   ’ A4jAž›ü!A$ÝC Aj–" (Aj"6 E
@   þ"E
 @ (Aj   ( (   ( ! A 6 @ E
  ^  7   7 A4j Ajý! (!AÝC   Å"   (Aj"6 E
@    þ" E
 @  (Aj     ( (   ( !  A 6   E
   ^AÝC!  ("AF
   6 @ 
   ( (   ("E
  Aj"6@ 
   ( (   ("E
  Aj"6 
   ( (   AÀ j$    Ù# A0k"$  A6 Aö’6  )7@@@@   AjÍ"E
   Ð"
  ((!  A6, Aö’6(  )(7@@   Ajû" 
 A !@@ ( "
  B 7  (! B 7  E
   6$  Aj6   ) 7    ïÑ!  ("E
   Aj"6 
     ( (   E
 ( !  A 6 @  E
   ("E
   Aj"6 
     ( (   AâC A0j$   æ# AÀ k"$ @@ ( "
 A !A ! Aj! (!  6  6  )7  Aj¿6 A 6< B 7( A6    AjA  A<jA  A jÒ!@ ( " AI
  A j   (0AA )(B€€ƒBˆ§Ó Ñ!  (! A 6@ E
  e AÀ j$   ô# Ak"$ @@@  
 A !    (Aj"6 E
@  å"E
   (Aj"6 E
  ("E
   Aj"6@ 
     ( (  @ 
     (Aj"6 E
@  é"E
   (Aj"6 E
  ("E
   Aj"6@ 
     ( (  @ 
 A ! A6 Aí„6  )7   ÿ! ("E
  Aj"6 
   ( (    ("E
   Aj"6 
     ( (   Aj$   ê
# AÐ k"$ A !@@ A J
   ( ! A68 Aó“64  )47  Aj€!	  ( !
 A60 Až›6,  ),7A !@ 
 Aj€"
E
 @ 
 æ
  
! 
("E
 
 Aj"6A ! 
  
 
( (  A !@@@@ 	
 A !	@ 	 æE
  	("
E
 	 
Aj"
6@ 
E
 A !	 	 	( (  A !	 A<j 	Þ@@  A<j—A H
   AÀ j—AH
 E
  E
@ (  F
   (Aj" 6  E
 ( !   6   E
   ("
E
   
Aj"
6 

     ( (   ( (kAu" AL
   AvAj6 (@!  A 6@@  E
   e (<! A ! A 6<  E
  e (@!
 A 6@@ 
E
  
e (<!
 A 6< 
E
  
e@ E
  ( (kAu" Av!@  AI
 A ! @    At"
¨6<@@ A<j —"A J
 @ E
 @ (  F
   (Aj"6 E
 ( !  6  E
  ("E
  Aj"6 
   ( (     6@ A N
 A!  (   j6   
Ar¢!A ! (<!
 A 6<@ 
E
  
e@ E
   Aj"  F
 AH
  (  j6 A !  ( !  A6( AÍ6$  )$7@   Aj€"

 A !  
(" 6H@@@  E
   6L A<j AÌ j AÈ j AÈ jç - DE
 
( 
(F
  Aj!
A !@  
 «" 6 @  E
    ("6H@@ E
   6L A<j AÌ j AÈ j AÈ jçA! - DAG
 A j  
   Ò"A G!  ("E
   Aj"6@ 
     ( (      Aj" 
( 
(kAuI
 A ! 
(" E
 
  Aj" 6  
  
 
( (   E
  (" E
   Aj" 6  
   ( (   	E
  	(" E
 	  Aj" 6  
  	 	( (   AÐ j$   9  A{A| j  lA Aj AI jAA jA  kqjAjA|qâCg# A k"$  B 7 A6  ( A  AjÕ! @ ("AI
  Aj  (AA )B€€ƒBˆ§Ó A j$   ¶# A0k"$    6,A !@@ A J
  A j  A,jÖ@@ - (" AG
  ($ (,6  - (Aq
  E
 A6 Až›6  )7@@ (, Ajÿ" E
   ("E
   Aj"6  (  (kAuAv! E
 A6 AÍ6  )7  (, ÿ" E
@@  (  (G
 A ! Aj!A !A !@@   ¬"E
    Õ! ("E
  j!  Aj"6 
   ( (   Aj"  (  (kAuI
   ("E
   Aj"6 
    ( (   A0j$   ¤~~~@ ( AK
 @@ )B€€øÿÿÿÿ ƒB R
  B€€7 Aj!A!A€•)!@ (" ( "G
  Aj!A !A€•)! AÐŸ ­"Að”)­"…BíéÄåè”‹Ù\~"B ˆ … …BíéÄåè”‹Ù\~"B ˆ …§ ­" …BíéÄåè”‹Ù\~"B ˆ … …BíéÄåè”‹Ù\~"B ˆ …§Aÿ q¶$" (j! ( Atj!A!   :    6   6     á# AÐ k"$  B€€€€p70  ( ! A6, Až›6(  )(7@@@@@@@@  Aj€"E
  ( (G
   ( ! A6$ AÍ6   ) 7@@  Ajÿ"E
  ("E
	  Aj"6 E
@ (0" G
  !  (Aj"6 E
  60 ! E
 ("E
  Aj"6 
  ( (   (0"
 A 6 B 7@ A68   A  Aj A0j A8jÒ!@ (8"AI
  A8j  (HAA )@B€€ƒBˆ§Ó 
 (0! 
   ( !A ! A 6 A8j A A  AjØ - HAG
@ (0" (@"F
 @ E
   (Aj"6 E
  60 E
  ("E
  Aj"6 
   ( (  @ E
  ("E
  Aj"6 
   ( (  @ (<"E
  ("E
  Aj"6 
   ( (  @ (8"E
  e (0! (4At"Aj! ( "
 B 78 (" E
   Aj" 6A !  
  ( (   (!	 B 78 	Aÿÿÿÿq"	E
   	6<  Aj68@   A8jÙ"E
  ("E
  Aj"6 
   ( (   (0 Aj ¸ A 6@ B 78   (0A  A8jÚ@ (8" (<"F
  !@@ ( " E
    A ¨6  Aj—! (! A 6@ E
  e@ AJ
 @@@ ( "
 A !A ! (! B 7 Aÿÿÿÿq"E
 Aj!  6  6  A  AjÛ"E
  ("E
  Aj"6 
   ( (     A¨6  Aj—! (! A 6@ E
  e AH
 @@@ ( "
 A !A ! (! B 7 Aÿÿÿÿq"E
 Aj!  6  6  A AjÛ" E
   ("E
   Aj"6 
     ( (   Aj" G
 @ E
   (@ kâCA!A !@ E
  (" E
   Aj" 6  
   ( (   (0!  A 60@  E
   ("E
   Aj"6 
     ( (  @ E
  (" E
   Aj" 6  
   ( (   AÐ j$   Ö# AÀ k"$ @@@ A!H
   A :   A :   A6< Až›68  )87@@  Aj€"E
 @  ( (kAuAv ( "j"I
   6    kAt"Ar¦"E
 A 60 B 7(   ¨6$ A(j A$j‘ ($! A 6$@ E
  e (,!  6,@ E
  ("E
  Aj"6 
   ( (   (0!  60@ E
  ("E
  Aj"6 
   ( (     ((6    (,6 (0!  A:    6   6 A6  AÍ6  )7@  Aj€"E
 @@ ( (F
  Aj!A !@@  «"E
       Ø ("E
  - !	  Aj"6@ 
   ( (   	Aÿq
 Aj" ( (kAuI
   A :   A :   (" E
   Aj" 6  
  ( (    A :   A :    A :   A :   (" E
   Aj" 6  
   ( (   AÀ j$  ¿~# A k"$ AÝC!   ("6@ E
   ( Aj6   ) "7  7  Aj AjÃ" (Aj"6@ E
  (! A 6@ E
  ë@    ¹"E
   (Aj" 6  E
 A j$   †	# AÐ k"$ A !@@@@ A J
   ( ! A6L Až›6H  )H7 @  A jÿ"E
  ("E
  Aj"6 
   ( (    ( !@@  G
  A6D Aó“6@  )@7  Aj€!@@ (" ("O
   6  Aj!	  ( "kAu" Aj"A€€€€O
@@  k"	Au"   KAÿÿÿÿ 	AüÿÿÿI"	
 A ! 	A€€€€O
 	AtÝC!   Atj" 6   	Atj! Aj!	@  F
 @ A|j" A|j"( 6   G
  (! ( !  6  	6  6  E
    kâC  	6@ 
 A! ("E
  Aj"6A! E
 A6< AÍ68  )87  Aj€"E
@@ ( (F
  Aj!
A !	@   	«"64@ E
 @ A4j  
 Ú"E
   ( ! A60 Aó“6,  ),7  Aj€!@@ (" ("O
   6  Aj!
  ( "kAu"Aj"A€€€€O
	@@  k"
Au"   KAÿÿÿÿ 
AüÿÿÿI"

 A ! 
A€€€€O
 
AtÝC!  Atj" 6   
Atj! Aj!
@  F
 @ A|j" A|j"( 6   G
  (! ( !  6  
6  6  E
    kâC  
6@ E
  ("E
  Aj"6 
   ( (   (4"
 A! ("E
  Aj"6@ 
   ( (   E
 A! 	Aj"	 ( (kAuI
 A ! ("E
  Aj"6 
  ( (   AÐ j$   Å6 ð ¿~# A k"$ AÝC!   ("6@ E
   ( Aj6   ) "7  7  Aj AjÃ" (Aj"6@ E
  (! A 6@ E
  ë@    ·"E
   (Aj" 6  E
 A j$   Ý# A k"$   ( ! A 6 Aj  A  AjØ@@ - "AG
 @ ("E
   (Aj"6 E
  ("Aj´  ´  (   AjA Ý (" E
   Aj" 6  
   ( (  @ - AG
  (! A 6@ E
  (" E
   Aj" 6  
   ( (   (! A 6@ E
  (" E
   Aj" 6  
   ( (   (! A 6 E
  e A j$   ¿	~~# A k"$ A !@@ A J
  A6œ Aó“6˜  )˜7@   AÀ j€! A 6” A 6@ E
  Aˆj Þ A”j Aˆj‘ Aj AŒj‘ (Œ! A 6Œ@ E
  e (ˆ! A 6ˆ E
  e A6„ Až›6€  )€78@@@   A8jÿ"E
 @  G
  E
  ( (F
 B !@@ (”" 
 A !B !	  AjA   5Bÿÿÿÿƒ"	§!@ ( " E
   (Aÿÿÿÿq"­B †  AjA  ­„!@@ 	 B ˆR
   § 	§At°7E
 Aj ßE
  (" 6|@  E
     ( Aj6   (”" 6x@  E
     ( Aj6 @ ( (kAM
 A ! @    At¨6ˆ@ Aˆj Aü j—AJ
  Aü j Aˆj@ Aˆj Aø j—AH
  Aø j Aˆj (ˆ! A 6ˆ@ E
  e  Aj"  ( (kAuAvI
 @@ (|" 
  B 7ˆ  (! B 7ˆ Aÿÿÿÿq"E
   6Œ   Aj6ˆ@ A  AˆjÛ" E
   ("E
   Aj"6 
     ( (  @@ (x" 
  B 7ˆ  (! B 7ˆ Aÿÿÿÿq"E
   6Œ   Aj6ˆ@ A AˆjÛ" E
   ("E
   Aj"6 
     ( (   (x!  A 6x@  E
   e (|!  A 6|  E
   e (" E
  F!   Aj" 6  E
 A6t AÍ6p  )p70@   A0j€"
 A !@@ ( (F
  Aj!
A !@@  «" E
 @     
ÝE
  A6l Až›6h  )h7(@@@@@   A(jˆ"E
  A6d Až›6`  )`7@   Ajÿ"( (G
 A!
 A6\ AÍ6X  )X7   Ajˆ
A !
 A6\ AÍ6X  )X7    A jˆE
 A6T AÍ6P  )P7   Ajÿ"("E
 ( (F!
  Aj"6@ 
   ( (   A G q
 E Asr
 ("E
  Aj"6 
   ( (   
E
   ´@ E
  ( (F
 @ A”j ß
  Aj ßE
  ("6|@ E
   ( Aj6   (”"6x@ E
   ( Aj6 @ ( (F
 A !@  ¬! A6L Aó“6H  )H7   ÿ! ("E
  Aj"6@ 
   ( (    A ¨6ˆ Aˆj Aü j—! (ˆ! A 6ˆ@ E
  e@ AJ
   A ¨6ˆ Aü j Aˆj‘ (ˆ! A 6ˆ E
  e  A¨6ˆ Aˆj Aø j—! (ˆ! A 6ˆ@ E
  e@ AH
   A¨6ˆ Aø j Aˆj‘ (ˆ! A 6ˆ E
  e ("E
  Aj"6@ 
   ( (   Aj" ( (kAuI
 @@ (|"
  B 7ˆ (! B 7ˆ Aÿÿÿÿq"E
   6Œ  Aj6ˆ@ A  AˆjÛ"E
  ("E
  Aj"6 
   ( (  @@ (x"
  B 7ˆ (! B 7ˆ Aÿÿÿÿq"E
   6Œ  Aj6ˆ@ A AˆjÛ"E
  ("E
  Aj"6 
   ( (   (x! A 6x@ E
  e (|! A 6| E
  e  ("E
   Aj"6A! 
    ( (    ("E
   Aj"6 
     ( (   Aj" ( (kAuI
 A ! (" E
   Aj" 6  
  ( (   (! A 6@ E
  e (”! A 6”@ E
  e E
  ("E
  Aj"6 
   ( (   A j$   Ð# Ak"$   A ¨6  A¨6@@ Aj Aj—AH
 @@ ("
  B 7  (! B 7  Aÿÿÿÿq"E
   6  Aj6 @ A  Û"E
  ("E
  Aj"6 
   ( (  @@ ("
  B 7  (! B 7  Aÿÿÿÿq"E
   6  Aj6 @ A Û"E
  ("E
  Aj"6 
   ( (    A ¨6  Aj ‘ ( ! A 6 @ E
  e  A¨6  Aj ‘ ( ! A 6  E
  e@ ( (kAu"AI
 @  Aj´ ( (kAu"AK
    ("6 @ E
   ( Aj6    ("6@ E
   ( Aj6  (! A 6 E
  e (! A 6@ E
  e Aj$  ~~B !@@  ( " 
 A !B !  AjA   5Bÿÿÿÿƒ"§!@ ( " E
   (Aÿÿÿÿq"­B †  AjA  ­„!A ! @  B ˆR
   § §At°7E!   ¿# A k"$   ( !A !  A 6 Aj  A  AjØ@@@@@ - "
  w 
  Aj‘ - "E
@ ("E
   (Aj" 6  E
 !  - Aq
 E
 (! A 6@ E
  ("E
  Aj"6 
   ( (   (! A 6@ E
  ("E
  Aj"6 
   ( (   (! A 6 E
  e A j$   t ­~~~# Ak"$  ( "Að”)s­BíéÄåè”‹Ù\~"B ˆ ­… …BíéÄåè”‹Ù\~"B ˆ …"Bÿ ƒB‚„ˆ À€~! §"Av /s! (!	 ( ! (!
A !@@@@ 
  q"j)  " …"B… Bÿýû÷ïß¿ÿ~|ƒB€‚„ˆ À€ƒ"P
 @ 	 z§Av j q"
Atj"(  F
 B| ƒ"B R
 @  B…B†ƒB€‚„ˆ À€ƒ"P
   6  z§Av j q6  )7  AÐŸ  ¹$! ( Atj!  (j!A! Aj" j!   
 
j!A !   :    6   6  Aj$ <~ ( "Að”)s­BíéÄåè”‹Ù\~"B ˆ ­… …BíéÄåè”‹Ù\~"B ˆ …§    AtÖ6
  AjA|qÝC¿~~~~@  ( "AI
   /!  (!  (! Av"	Aøÿÿÿq!
Að”)­!A !@  j)  !
  j" B€‚„ˆ À€7     	jAjB€‚„ˆ À€7  @ 
B€‚„ˆ À€ƒ"
B€‚„ˆ À€Q
  
B€‚„ˆ À€…!
@  
z§Av r" Atj"5 " …BíéÄåè”‹Ù\~"B ˆ … …BíéÄåè”‹Ù\~"B ˆ …§"Aÿ q!@@@@ 
   Av s"k"q
  Aq j q!   	q  O
   q"j)  B€‚„ˆ À€ƒ"P
 z§Av j!    j :     Atj ( 6          
B| 
ƒ"
PE
  Aj" 	I
 î# A k"$    ("6@@@ E
   6 Aj Aj Aj AjçA! - AG
 Aj  À"( "( E
@@ ("  ("F
 @   ( ("6@ E
   6 Aj Aj Aj AjçA! - AG
  Aj"  G
 A ! Â A j$   “~@@@ ( "( AK
 @@@ )B€€øÿÿÿÿ ƒB R
  B€€7 Aj!A€•)! (" ( "F
 AüŸ Að”)s­BíéÄåè”‹Ù\~"B ˆ …§ Að”)s­BíéÄåè”‹Ù\~"B ˆ …§Aÿ q¶$" (j! ( Atj!  A:    6   6   A :    Aj6  A€•)6     è  - AqE
  ( ( 6 –~~~# Ak"$  ( "Að”)s­BíéÄåè”‹Ù\~"B ˆ …"Bÿ ƒB‚„ˆ À€~! §"Av /s! (!	 ( ! (!
A !@@@@ 
  q"j)  " …"B… Bÿýû÷ïß¿ÿ~|ƒB€‚„ˆ À€ƒ"P
 @ 	 z§Av j q"
Atj"(  F
 B| ƒ"B R
 @  B…B†ƒB€‚„ˆ À€ƒ"P
   6  z§Av j q6  )7  AüŸ  ¹$! ( Atj!  (j!A! Aj" j!   
 
j!A !   :    6   6  Aj$ #~ ( Að”)s­BíéÄåè”‹Ù\~"B ˆ …§¥~~~@  ( "AI
   /!  (!  (! Av"	Aøÿÿÿq!
Að”)­!A !@  j)  !
  j" B€‚„ˆ À€7     	jAjB€‚„ˆ À€7  @ 
B€‚„ˆ À€ƒ"
B€‚„ˆ À€Q
  
B€‚„ˆ À€…!
@  
z§Av r" Atj"5  …BíéÄåè”‹Ù\~"B ˆ …§"Aÿ q!@@@@ 
   Av s"k"q
  Aq j q!   	q  O
   q"j)  B€‚„ˆ À€ƒ"P
 z§Av j!    j :     Atj ( 6          
B| 
ƒ"
PE
  Aj" 	I
 ó# Ak"$   A 6  B 7  A :   A 6 @ AŸ€€€xjA €€€xI
  AH
  Aàÿÿÿ AjAàÿÿÿq"nK
    6   6   Av"6  AX!@  - "AÿF
 @ AG
   ( !   6  E
 J A )° 7 Aj   Aj Atj(    A:    6  Aj$     A 6  B 7  A :   A 6 @  rA H
  Aüÿÿÿ K
  Aq
  At" H
  Aàÿÿÿ nJ
    6   6   6 ( "E
    6   œ~# Ak"$   A :   A 6    (6   (6   (6@ - "AÿF
   Aj6 A )¨ "7@ Aj  Aj Atj(  E
   (  (AX!@@@  - "AÿF
 @ AG
   ( !   6  E
 J  - "AÿG
 A )° 7 Aj   Aj Atj(    A:    6 A!  Aj6  7 Aj   Aj Atj(  ! - "AÿF
  Aj6  7 Aj  Aj Atj(  !  (  (l"E
    Ö6 Aj$   ¯ T# Ak"$ @  - "AÿF
  A )° 7 Aj   Aj Atj(    Aÿ:  Aj$      AjAÿÿI AjAÿÿIqò~# Ak"$ @  - "AÿF
   Aj6A ! A )¨ "7@ Aj   Aj Atj(  E
 A ! A H
    (N
 A ! A H
    (N
   - "AÿF
  Aj6  7 Aj   Aj Atj(    ( lj" E
    Avj-   AsAqvAq! Aj$  ¯ ú~# Ak"$ @  - "AÿF
   Aj6 A )¨ "7 Aj   Aj Atj(  !@ A H
  E
    (N
  A H
    (N
   - "AÿF
  Aj6  7 Aj   Aj Atj(    ( lj" E
    Avj"   -  " A AsAq"tr  A~ wq :   Aj$ ¯ ¶~# Ak"$ @  - "AÿF
   Aj6 A )¨ "7@ Aj   Aj Atj(  E
  A H
    (N
   - "AÿF
  Aj6  7 Aj   Aj Atj(    (" lj"E
 @@ A H
    (N
   - "AÿF
  Aj6  7 Aj   Aj Atj(    (" lj" 
 E
 A  Ø6 E
     Ö6 Aj$ ¯ Ó~# Ak"$ @@  - "AÿF
   Aj6 A )¨ "7@ Aj   Aj Atj(  E
   - "AÿF
  Aj6  7 Aj   Aj Atj(  !  ("A H
  4 ­~"B€€€€Z
 P
  A  k §Ø6 Aj$ ¯ Œ# A k"$ @  - "AÿF
   Aj6A ! A )¨ 7@ Aj   Aj Atj(  E
    )7 B 7       Ajõ! A j$  ¯ £#~~# Ak"$ A !@ Aÿÿ¿jAÿÿÿ~I
  Aÿÿ¿jAÿÿÿ~I
 A ! (" k"	 ( ("
k"  	A  	 s  sqAJ"J  " AuA  kq"
L
  (" k"	 ( ( k"  	A  	 s  sqAJ"J  " AuA  kq"L
 @@@  - "	AÿF
   Aj6 A )¨ "7 Aj   Aj 	Atj(  !	 (  jA m!  - "AÿF
   (!  Aj6  7 Aj   Aj Atj(  !  ("A H
  ("¬ ­~"B€€€€Z
 A`m! - "AÿF
   
k A  A J"j!AA   k A  A J"j"kt!A v! Aq! Aq! 	  
 
jlj Atj!
  §j!  Aj6  7 Aj  Aj Atj(   ( lj AvAüÿÿÿ qj!@ Aj sAK
   q!	@ Aj sAK
 @  M
 @  H
 A!  k! 	As!@ 
 I! 
 O
 (  "At A€þqAtr AvA€þq Avrr! 
(  "At A€þqAtr AvA€þq Avrr t!A !@@@@@@    r 	q  qr!  r q!  s 	q  qr! 	  sAsq  qr!  q  	qr!  At A€þqAtr AvA€þq Avrr6    (j! 
  (j!
 Aj" G
  @  H
 A!  k! 	As!@ 
 I! 
 O
 (  "At A€þqAtr AvA€þq Avrr! 
(  "At A€þqAtr AvA€þq Avrr v!A !@@@@@@    r 	q  qr!  r q!  s 	q  qr! 	  sAsq  qr!  q  	qr!  At A€þqAtr AvA€þq Avrr6    (j! 
  (j!
 Aj" G
  @  H
 A!A   k"k! 	As!
@ 
 I! 
 O
 
Aj(  "At A€þqAtr AvA€þq Avrr v 
(  "At A€þqAtr AvA€þq Avrr tr! (  "At A€þqAtr AvA€þq Avrr!A !@@@@@@    r 	q  
qr!  
r q!  s 	q  
qr! 	  sAsq  
qr!  
q  	qr!  At A€þqAtr AvA€þq Avrr6    (j! 
  (j!
 Aj" G
   Aq!  Atj!  M
@  H
 A! Au AjAvk!A   k"
k! As! As!@ 
 I! 
 O
 ! 
!@ E
  
Aj"(  "At A€þqAtr AvA€þq Avrr v 
(  "At A€þqAtr AvA€þq Avrr 
tr! (  "At A€þqAtr AvA€þq Avrr!A !@@@@@@    r q  qr!  r q!  s q  qr!   sAsq  qr!  q  qr!  At A€þqAtr AvA€þq Avrr6   Aj!A !	@ A L
 @ (  ! Aj"(  "At A€þqAtr AvA€þq Avrr v At A€þqAtr AvA€þq Avrr 
tr! (  "At A€þqAtr AvA€þq Avrr!A !@@@@@@    r!  q!  s!  sAs! !  At A€þqAtr AvA€þq Avrr6   Aj! 	Aj"	 G
 @ E
  (  "At A€þqAtr AvA€þq Avrr 
t!A !A !	@ Aj" 
 jO
  (  "At A€þqAtr AvA€þq Avrr!	 (  "At A€þqAtr AvA€þq Avrr! 	 v r!@@@@@@    r q  qr!  r q!  s q  qr!   sAsq  qr!  q  qr!  At A€þqAtr AvA€þq Avrr6    (j! 
  (j!
 Aj" G
  ¯ @  G
 @  H
 A! Au AjAvk!
 As! As!@ 
 I! 
 O
 
! !@ E
  (  "At A€þqAtr AvA€þq Avrr! 
(  "At A€þqAtr AvA€þq Avrr!A !@@@@@@    r q  qr!  r q!  s q  qr!   sAsq  qr!  q  qr!  At A€þqAtr AvA€þq Avrr6   Aj! 
Aj!A !@ 
A L
 @ (  "At A€þqAtr AvA€þq Avrr! (  "At A€þqAtr AvA€þq Avrr!	A !@@@@@@    	r!  	q!  	s!  	sAs! 	!  At A€þqAtr AvA€þq Avrr6   Aj! Aj! Aj" 
G
 @ E
  (  "At A€þqAtr AvA€þq Avrr! (  "At A€þqAtr AvA€þq Avrr!A !@@@@@@    r q  qr!  r q!  s q  qr!   sAsq  qr!  q  qr!  At A€þqAtr AvA€þq Avrr6    (j! 
  (j!
 Aj" G
  @  H
 A! Au AjAvk!A   k"
k! As! As!@ 
 I! 
 O
 !@ E
  (  "At A€þqAtr AvA€þq Avrr! 
(  "At A€þqAtr AvA€þq Avrr 
v!A !@@@@@@    r q  qr!  r q!  s q  qr!   sAsq  qr!  q  qr!  At A€þqAtr AvA€þq Avrr6   Aj!A !	 
!@ A L
 @ (  ! Aj"(  "At A€þqAtr AvA€þq Avrr 
v At A€þqAtr AvA€þq Avrr tr! (  "At A€þqAtr AvA€þq Avrr!A !@@@@@@    r!  q!  s!  sAs! !  At A€þqAtr AvA€þq Avrr6   Aj! 	Aj"	 G
 @ E
  (  "At A€þqAtr AvA€þq Avrr t!A !A !	@ Aj" 
 jO
  (  "At A€þqAtr AvA€þq Avrr!	 (  "At A€þqAtr AvA€þq Avrr! 	 
v r!@@@@@@    r q  qr!  r q!  s q  qr!   sAsq  qr!  q  qr!  At A€þqAtr AvA€þq Avrr6    (j! 
  (j!
 Aj" G
  Aj$  Ñ~# A k"$ @  - "AÿF
   Aj6A ! A )¨ "7@ Aj   Aj Atj(  E
  - "AÿF
  Aj6  7@ Aj  Aj Atj(  
 A !  )7 B 7       Ajõ! A j$  ¯ µ~# Ak"$ @  - "AÿF
   Aj6A ! A )¨ "	7@ Aj   Aj Atj(  E
  - "AÿF
  Aj6  	7 Aj  Aj Atj(  E
        õ! Aj$  ¯ ø~# Ak"$ @AÝC  ë"- "AÿF
   Aj6 A )¨ "7@ Aj  Aj Atj(  E
   - "AÿF
  Aj6  7 Aj   Aj Atj(  ! A H
  E
  A H
    (N
    (N
 @ Aq
        ù       ú Aj$  ¯ ¥~# Ak"$  Am!@@  ( k" ("  H"	AH
   ( k" ("  H!
A !A )¨ !@ - "AÿF
  Aj6  7 Aj  Aj Atj(  !  - "AÿF
 (!
  Aj6  7 Aj   Aj Atj(  !@ 
E
   
 lj   (  jlj j 
Ö6 Aj" 	G
  Aj$ ¯ ï
~# Ak"$  A m!@@  ( k" ("	  	H"
AH
   ( At"k" ("	  	H!A  Aq"
k!A !A )¨ !@  - "AÿF
  Aj6  7 Aj   Aj Atj(  ! - "AÿF
  (!  Aj6  7@ Aj  Aj Atj(   ( lj"	 	 j"O
     jlj"  (j!  j!@ (  "At A€þqAtr AvA€þq Avrr 
t!@ Aj" O
  Aj(  "At A€þqAtr AvA€þq Avrr v r! 	 At A€þqAtr AvA€þq Avrr6   ! 	Aj"	 I
  Aj" 
G
  Aj$ ¯ Ò
~~~# Ak"$ @  - "AÿF
   Aj6 A )¨ "7@@ Aj   Aj Atj(  E
    ("L
  Aüÿÿÿ   ("mJ
  A H
 ¬" ­~"B€€€€Z
  ­~"B€€€€Z
 §!	 §!@@@  - "
AG
   ( !
  A 6  Aj  A )° "B ˆ§   A :   A 6  
 AY!
@@  - "AÿF
 @ AG
   ( !   
6  E
 J  7 Aj   Aj Atj(    A:    
6 A!
 
AÿF
  Aj6  7 Aj   Aj 
Atj(  ! AW!
@@@  - "AÿF
 @ AG
   ( !   
6  E
 J  - "
AÿG
 A )° 7 Aj   Aj Atj(    A:    
6 A!
  Aj6  7 Aj   Aj 
Atj(  !
 E
  
  	Ö6  - "
AÿF
  Aj6  7 Aj   Aj 
Atj(  !
@  F
  
 	jA  k  	kØ6   6 Aj$  ¯   (   ( 	  A 6  ( ! A 6 @ E
  J3   B 7   A jA 6   AjB 7   AjB 7   AjB 7   ¶@  ("E
  !@   ("F
 @ A|j"( ! A 6 @ E
  îAâC  G
   (!   6   (  kâC@  ("E
    6   ( kâC@  ( "E
    6   ( kâC  Å# Ak"$ A$ÝC"B 7  A jA 6  Aj"B 7  AjB 7  AjB 7 @  ("  ("F
 @@@ ( "
 A !AÝC í!  6@@ (" ( O
  A 6  6  Aj!  Ajƒ!  6 (! A 6@ E
  îAâC Aj" G
 @   F
    ( "  ("  kAu„ Aj  ("  ("  kAu„ Aj$  å@@  (  ( "kAu"Aj"A€€€€O
 A !@  ( k"Au"   KAÿÿÿÿ AüÿÿÿI"E
  A€€€€O
 AtÝC! ( ! A 6   Atj" 6   Atj! Aj!@  ("  ( "F
 @ A|j"( ! A 6  A|j" 6   G
   (!  ( !   6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  îAâC  G
 @ E
    kâC Å6 ð Õ@   ("  ( "k"AuK
 @   (" k"AuM
   j!@  F
    ×6  (!  k!@  F
    ×6    j6  k!@  F
    ×6    j6@ E
    6  âCA !  A 6  B 7 @ A€€€€O
  Au"   KAÿÿÿÿ AøÿÿÿI"A€€€€O
    At"ÝC"6   6     j6  k!@  F
    Ö6    j6Å6    A 6    6    6   €@  (E
   ("(   ( ("6  ( 6   A 6   F
 @ (! A 6 (!@ E
  A$âC A âC !   G
    @  
  @ E
    ˆ  ö# Ak"$   (6  ("6  ("6 Aj! Aj!@@ 
   6  6 B 7  Aj6   Aj‰ Aj (Š ( ! A 6    ‹  (Š ( !  A 6 @@  E
   ("E
   Aj"6 
     ( (   AâC Aj$  ÷	# Ak"$ @ (E
   Aj!@  (
    (Š   ( 6   ("6   ("6  Aj!@ 
   6   6 B 7  Aj6  Aj!@  ("  Aj"F
  ( " F
 @@@ ("	 ("G
 @ - AG
  - AG
  - AG
  A: @@ ("
E
 @ 
"( "

  @  ("( G!
 ! 

 @ ("E
 @ "( "
  !@  ("( F!
 ! 
E
  ! !@ 	 O
 @   Aj Aj Aj"
”"	( 
 A(ÝC"A j 
Aj) 7  Aj 
Aj) 7   
) 7 (!
 B 7   
6 	 6 @ ( ( "
E
   
6  	( ! ( È  (Aj6@ ("E
 @ "( "
  @  ("( F!
 ! 
E
  ! ! ( "E
@   ( 	I"
!  
Atj( "
   F
  G
 @  F
 @@   Aj Aj Aj"”"
( 
 A(ÝC"A j Aj) 7  Aj Aj) 7   ) 7 (! B 7   6 
 6 @ ( ( "E
   6  
( ! ( È  (Aj6@@ ("E
 @ "( "
  @  ("( G! ! 
  !  G
    (Š   ( 6   ("6   ("6@ 
   6   6 B 7  6  Aj$ % @ E
    ( Š   (Š A(âC# AÀ k"$ @@ E
 @  ( 
    6  A$jAå²ü!  ( ! A6< Aå²68  )87    Aj‹ ( ! A 6 @ E
  ^ A$jAÐ‡ü!  ( ! A64 AÐ‡60  )07    Aj‹ ( ! A 6 @ E
  ^ A$j ‰@ ($" (("F
 @@@@ ( "
 A !A ! (! B 7 E
 Aj!  6   6  ( !  )7     ‹ Aj" G
  ($!@ E
  !@  (("F
 @ A|j"( ! A 6 @ E
  ^  G
  ($!  6(  (, kâC ("E
  Aj"6 
   ( (   AÀ j$     B 7  B 7     Aj6  #   B 7   6   6     Aj6  W  Aj  (Š  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     ä@ A€€€O
  A€€€O
   Aj"! !@@  ("E
 @@  "("O
  ! ( "
@  I
  ! ("
  Aj!A(ÝC"B 7  6  6 B 7  A jB 7   6  !@  (( "E
    6 ( !  ( È    (Aj6@ /
  - 
   6$  6  A:  A ; !@@ ( "E
 @@  "("O
  ! ( "
@  I
  ! ("
  Aj!A(ÝC"B 7  6  6 B 7  A jB 7   6  !@  (( "E
    6 ( !  ( È    (Aj6 A:  ‘@ A€€€O
 @@@  ("
   Aj"!@@  "("O
  ! ( "
@  I
  ! ("
  Aj!A(ÝC"B 7  6  6 B 7  A jB 7   6  !@  (( "E
    6 ( !  ( È    (Aj6@ / K
  A:   7   ;  -  r:  ÷@ A€€€O
 @@@  ("
   Aj"!@@  "("O
  ! ( "
@  I
  ! ("
  Aj!A(ÝC"B 7  6  6 B 7  A jB 7   6  !@  (( "E
    6 ( !  ( È    (Aj6 B 7   ; A :  M  ( !   6 @@ E
  ("E
  Aj"6 
   ( (     6 bA !@  ("E
   Aj"! @    ( I"!   Atj( "
    F
 A   Aj   (I! ƒ@@   Aj"F
  ( " ("O
 ( ! !@@   ( F
 @@ 
  ! @    ("( F! !  
   ! @  "(" 
  ( ( "O
@ 
   6    6  Aj@ ( " 
   6   !@@@   "(" O
  ! ( " 
   O
 Aj! (" 
   6  @  O
 @@ ("
  ! @    ("( G! !  
   ! @  "( " 
 @@  F
   (O
@ 
   6  Aj  6  @ ( " 
   6   !@@@   "(" O
  ! ( " 
   O
 Aj! (" 
   6    6   6  £  Aj!  Aj!  (!@ 
   Š   6  B 7@@@@ E
  ! !@   ( I"!  Atj( "
 @  F
 @ !@@ ("E
 @ "( "
  @  ("( G! ! 
 @ (  G
   6     (Aj6  ( ï A(âC !  G
  ( ! Aj!@ 
  ! ! !@   ( I"!  Atj( "
 @  F
   (O
@@  "("O
  ( "
 !@  I
  ! ("
  Aj! Aj! !A(ÝC"B 7  6  6 B 7  A jB 7   6  !@ ( ( "E
   6  ( !  ( È    (Aj6 B 7 &    6    - :    - :  A ;  ‚  ( " -   - r:   ( " -   - r:   ( !  A 6 @@ E
   ( Apj( j"("E
  Aj"6 
   ( (     \   Að 6   A 6$  A 6   6   6  AÌ 6   AjA 6      ( Alj( j" ( (  7  q  A 6  AÌ 6   Að 6   (!  B 7@@ E
   ( Apj( j"("E
  Aj"6 
   ( (         ( Atj( j" AÌ 6   Að 6   B 7  (!  A 6@@ E
   ( Apj( j"("E
  Aj"6 
   ( (     v  A 6  AÌ 6   Að 6   (!  B 7@@ E
   ( Apj( j"("E
  Aj"6 
   ( (    A(âC „    ( Atj( j" AÌ 6   Að 6   B 7  (!  A 6@@ E
   ( Apj( j"("E
  Aj"6 
   ( (    A(âC ¼~~# Ak"$ @@ B S
   ("­|" B…ƒB S
    )U
 @  - 
   ("E
     ( ( 
   A:  E
  ("E
  B€üÿÿÿÿÿÿÿ ƒB€|B€ B U" B S"  )"  S"BÿÿÿÿV
  B€üÿÿÿÿÿÿÿ ƒ"}"BÿÿÿÿV
   § ( (*   (!  ) "7 ( (!  7 A!     
  A; E
   ("E
   B€üÿÿÿÿÿÿÿ ƒB€|B€ B U" B S"  )"  S"BÿÿÿÿV
   B€üÿÿÿÿÿÿÿ ƒ"}"BÿÿÿÿV
    § ( (* A ! Aj$     )§~~~~A!@   )"U
 @  ­|" B…"ƒB S
  B€|" B…ƒB S
     S"BÿÿÿÿV
   }"BÿÿÿÿV
   ƒB S
   - 
  ("E
   §" ( ( 
  A:  E
   ("E
   ƒB S
  B€üÿÿƒB€|"  )"  S"BÿÿÿÿV
   B€üÿÿÿÿÿÿÿ ƒB  B U"}"BÿÿÿÿV
    § ( (* A ! ¿~~  )!@@@  - "
  BÿÿÿÿV
 @  ("
   A: A   B  § ( ( ":  E
A E
A  )!@ BÿÿÿÿV
   A:  P
   (" E
   B  B€üÿÿƒB€|"   T§  ( (* A "   A 6  B 7   :   A€¡6      Ô   ÔAâC A2AÝC!  - !  A 6 B7   :  A€¡6  1# Ak"$  AjAžÉA®Ì  - ü( !  Aj$      -     AžÉ‰:    Á# A0k"$  A6( Aø£6$  )$7A !@  Aj°E
 @ A,jAžÉA®Ì  - ü( " E
   (! B 7@ E
   6    Aj6  )7  Aj°!  ^ B 7  B 7  °! A0j$     A 6  B 7  Aä¡6    A"AÝC"A 6 B7 Aä¡6     9# Ak"$  A6 Aî´6  )7   °! Aj$     ÔAâC'   Aø¢6  A 6  AÔ¢6    ) 7          ( Atj( j	   AâC     ( Atj( jAâC   5~A !@@ B S
  ("E
   ­|"BÿÿÿÿV
   B…ƒB S
   (" §I
   §"I
   kK
 (   ( j ×6A!  ¶# Ak"$   A¸£6  A 6   A 6  B 7  A”£6  ( ! A 6    6 (! A 6   6AÝC!  (!  6  6  )7   ±" (Aj"6@ 
     6 Aj$     (!  A 6@@ E
  ("E
  Aj"6 
  Aj" ( (    (!  A 6@ E
  J@  ("E
    6 J       ( Atj( j" (!  A 6@@ E
  ("E
  Aj"6 
  Aj" ( (    (!  A 6@ E
  J@  ("E
    6 J   †  (!  A 6@@ E
  ("E
  Aj"6 
  Aj" ( (    (!  A 6@ E
  J@  ("E
    6 J  A$âC ”    ( Atj( j" (!  A 6@@ E
  ("E
  Aj"6 
  Aj" ( (    (!  A 6@ E
  J@  ("E
    6 J  A$âC 
   (¶=~# Ak"$   (!   ) "7   7    ·!  Aj$   ©~A(ÝC A ˜" ( Apj( j" (Aj"6@ 
    B 7   6   ( (  !  B 7   7  A jB 7   A(jB 7   A0jB 7   A8jB€€€€€À 7   AÀ jA AØ6  A 6Ä  t    7   6   ( (  !  B 7   7  A jB 7   A(jB 7   A0jB 7   A8jB€€€€€À 7   AÀ jA AØ6  A 6Ä  “  A 6Ä@  ($"E
    6( J  ( !  A 6 @ E
  ë  ( !  A 6 @@ E
   ( Apj( j"("E
  Aj"6 
   ( (     $~  )!   7   Ã!   7 ë
~~~# Ak"$ A !@  )"  )  )|"W
 @@@@  )0" W
   (("  ($"k!	    (("  ($"k"	­|S
@@  }§"
 
  (<"  ­|" B…ƒB S  U"
 	M
   A$j 
 	kÜ  ($!  ((! 
 	O
     
j"6(  ( !	   k6  6 	( (!  )7  	    E
   70  ($! !    }§j-  :      )B|7A!    ($6( Aj$  
   )  )}õ	~~~# Ak"$ A !@  ) |"  )"Y
 @@  )0" U
     ((  ($"k­|S
  B||B  BÿU"W
@@  }§"   (<"  ­|"	 B…ƒB S 	 U"  (("
  ($"k"M
   A$j  kÜ  ($!  ((!
  O
     j"
6(  ( !  
 k6  6 ( (!  )7 @     
     ($6(   70  U
    ((  ($"k­|Y
    }§j-  :  A! Aj$  v~~~# Ak"$   ( !  ) "7 ( (!  )!  )!  7 @    |  "E
     ) B ˆ|7 Aj$  Ÿ# Ak"$   A 68  ÈA!@   AjÃE
   AÀ j!@@A€œ - "j-  "AÄ F
 A!@@  (8"AÿK
    Aj68  j :   AÿqAÎ F q!   AjÃE
A€œ - "j-  "A¼j                        (8"Aj68  j :  A !@@@ AQj    AjÃE
@@@A€œ - "j-  A²j        )B|7@  (8"AÿK
    Aj68  j :     AjÃ
     AjÃE
@ - A<G
     (8"Aj68  jA<:      )B|7   AjÃE
@ - A>G
     (8"Aj68  jA>:      )B|7    )B|7 Aj$  ™# Ak"$ @@  (ÄE
   É   AjÃE
 @@A€œ - "j-  A× G
    AjÃ
@ A%G
 @   AjÃE
 - Avj        )B|7 Aj$ ˆ# Ak"$ @@@   AjÃE
 A !@ - !@@@@@@  @A€œ j-  A× G
 A !AA A%F!AAA A%F AÅ F!AA AÏ F!AA AÆ F!@@ A
G
    AjÃ! - ! E
  AÿqA
F
 A
! A
:     )B|7A! AÿqAvj    (!@@  (Ä"(" ("O
   6  Aj!  ( "kAu"Aj"	A€€€€O
@@  k"
Au" 	  	KAÿÿÿÿ 
AüÿÿÿI"

 A ! 
A€€€€O
 
AtÝC!  Atj"	 6   
Atj!
 	Aj!@  F
 @ 	A|j"	 A|j"( 6   G
  (! ( !  
6  6  	6  E
    kâC  6A!@A A   Aÿq"A
F A
F"AF
    AjÃ
    )B|7 Aj$ Å6 ð ¼# Ak"$ A !@   AjÃE
  A 6A !A !A !@ - "Aøq! À!@@@@@@@@@@@@@  @ AXj @ A
G
 A ! AÜ G

 A0F
@ A0G
  APjA  ’7!A!
A
!@@@@@@@@ Ažj A! AvjA
!A	!A!A! ! Aj †A !	A! APjA  ’7 Atr! A0G
 A ! Aj APjA  ’7 Atr"À† Aj À†@ AXj  AÜ F
@ 
  ("
A ! Aj! Aj! Aj †A !  ( Aj6  (! A 6 E
 ^A!   AjÃ
    AjÃ (! Aj$  ¸# Ak"$ A !  AjÃ!  A 6  B 7 @@ E
  - "AÿqA>F
 A!A !@@ À"A€q
  “7E
 AIAP A`j  AŸjAI"ÀA9J j!@@ AqE
  At!  j!@@   ("O
   :   Aj!   ( "k"	Aj"AL
@@  k"At"
  
 KAÿÿÿÿ AÿÿÿÿI"
 A ! AV!  	j" :    j!
@@  G
  !A !
 ! !@ 	Aq"	E
 @ Aj" Aj"-  :   
Aj"
 	G
 @  kA|K
 @ Aj Aj-  :   A~j A~j-  :   A}j A}j-  :   A|j" A|j"-  :    G
   ( ! Aj!   
6   6  E
  J   6 As!@  AjÃE
  - "AÿqA>G
 Aq
 @@   ("O
   :   Aj!   ( "k"
Aj"AL
@@  k"At"   KAÿÿÿÿ AÿÿÿÿI"
 A !	 AV!	 	 
j" :   	 j!@@  G
  !	 ! !@ 
Aq"E
 A !
 ! !@ Aj" Aj"-  :   
Aj"
 G
 @  kA|K
 @ Aj Aj-  :   A~j A~j-  :   A}j A}j-  :   A|j" A|j"-  :    G
   ( ! Aj!   6   	6  E
  J   6 Aj$ Å6 b# Ak"$ @@   AjÃE
 - Aÿq"A
F
 A
G
    AjÃ - A
F
     )B|7 Aj$ ´# A k"$ @@ ( "E
   ( Apj( j" (Aj"6 E
 Aj –! Ç!@ ( "E
   ( Apj( j" (Aj"6 E
  ( Apj( j"("E
  -  - r!  Aj"6@ 
   ( (  A !A !@ Aq
  (8"A‚O
 AÀ jA  !  6  6  )7   Ajý :  — A j$  5@  ( " E
     ( Apj( j" (Aj"6 
    9~# Ak"$   )! Aj  Í (!   7 Aj$  '# Ak"$  Aj  Í (!  Aj$   ~    )"   S7¤# Ak"$ @@  ( "E
   ( Apj( j" (Aj"6 E
 Aj –!   AÓ!@  ( " E
     ( Apj( j" (Aj"6 E
    ( Apj( j"("E
   -   - r!   Aj"6@ 
   ( (  @@  Aq
  ! A !  E
  ("E
  Aj"6 
   ( (   — Aj$    æ~# Aà k"$ A A (Äƒl"Aj"6Äƒl  6\A !@@ AÀ J
   )! AÔ j  Í@ (T"
 A !A !@ (E
   Aj!@ - XAG
   )! AÌ j  Í@@ - P
 @@ (T"
 A !A !  AjA  (" !AÝC!   6<  68  )87  Ajó" (Aj" 6  
 AÄ j  Í@@ AÄ jA þ‰
 @@ (T"
 A !A !  AjA  (" !AÝC!   6<  68  )87  Ajó" (Aj" 6  E
  (T"AjA²œ —" 68A !A !  AF
  AÜ j A8jÔ! (D!  A 6D  E
   ^ (L!  A 6L@  E
   ^ E
  7 @@ AÔ jAžÉ‰
  AÔ jA®Ì‰E
 AÔ jAžÉ‰!AÝC ¡" (Aj" 6  E
@ AÔ jAï´‰E
 Õ!@ AÔ jAñš‰E
    Ê68  A j A8jÖ! (8!  A 68  E
  ^@ AÔ jAþ‡‰E
  A8j  Ë A 6L  A j A8j AÌ j×! (8" E
   6<  J@@ AÔ jAó‰E
 Ø!@   AÓ"E
 @@@ ó
   » ("E
  Aj"6 
   ( (     AÓ"
 @ E
  !@  - @AÿqAÝ G
  !A ! E
 ("E
  Aj" 6A !  
  ( (   (T"E
 (E
@ - A/G
   (8"A‚O
  A j!@@@@   B 70 B 70  Aj64   AÁ j60  )07  AjÏ68  A8jÙ! (8!  A 68  E
  ^@ AÔ jAý‡‰E
   A jÚ!	 A8j  Í@ (8"E
 @A!@ ("E
  ) !A! A8jAº‡‰
 @ A8jAÑ¿‰E
    ­}7  (8"E
 ("
E
A! - A/G
   
6,  Aj6(  )(7   A jÏ"6L@@ 
 A! (E!A!@@ E
  
@   AÓ"
  
  ÌA!@ (L"E
  (AI
  ó
   AÌ jA6D 	 AÄ j ‹ (D! A 6D@ E
  ^A ! ("E
  Aj"
6A ! 

   ( (   (L! A 6L E
  ^ (8! A 68@ E
  ^@@    )! A8j  Í A8jAÒ´‰! (8! A 68@ E
  ^@ E
    	Û!  7  	! A8j  Í (8"
 A ! 	E
 	("E
 	 Aj" 6A !  
 	 	( (  A ! AÔ jAº‡‰E
  7 A ! (T!  A 6T  E
   ^A  6Äƒl Aà j$   - AÝC  (  ( Å" (Aj" 6@  
   'AÝC«"   (Aj"6@ 
    €# Ak"$ AÝC!   ( " 6@  E
     ( Aj6   Aj Â"   (Aj"6@ E
  (! A 6@ E
  ë Aj$    ³# A k"$ AÝC!   ( " 6@  E
     ( Aj6  (!   ( "6    k6 ( !   )7  Aj Aj  Á"   (Aj"6@ E
  (! A 6@ E
  ë A j$    'A$ÝC•"   (Aj"6@ 
    €# Ak"$ AÝC!   ( " 6@  E
     ( Aj6   Aj ×"   (Aj"6@ E
  (! A 6@ E
  ë Aj$    'A$ÝC  â"   (Aj"6@ 
    €~~~~# A0k"$  A6 A÷Á6  )7B!A !@@  Ajï"E
 @ ï"E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   E
  ù¬! !@@   A$jÃE
 - $Aÿq"A
F
 A
G
    A$jÃ - $A
F
     )B|7  )!@@@@@@@@@@ BS
   |"	 … 	 …ƒB S
 	  )Y
 !	@  ( "E
   ( Apj( j" (Aj"6 E
  )!	  	  )| §Ÿ!  ( Apj( j"("E
  Aj"6@ 
   ( (  A ! E
	@  ( "E
   ( Apj( j" (Aj"6 E
  )!	  )!
A ÝC"A6  7  	 
|7  6 AÔ£6  Aø£6    )"
 	 |" 
 S7A ! B S
@  ( "E
   ( Apj( j" (Aj"6 E
	 A$j –!  )! A : / A : #   A/jÃ   B|7   A#jÃ - #! - /!  B 7@  AÈ jA ;    B A
F" A
Fr­"	 AÿqA
F 	 |7  Ç@  ( "E
   ( Apj( j" (Aj"6 E
	  ( Apj( j"("E
 -  - r!  Aj"6@ 
   ( (   Aq
  AÀ jA¡´A	°7E
@ E
   ( Apj( j"("E
	  Aj"6 
   ( (      )"   S7 —A !  Ü"B S
  }"BS
    )"	  	 S"7@  ( "E
   ( Apj( j" (Aj"6 E
  )!    )| §Ÿ!  ( Apj( j"("E
  Aj"6@ 
   ( (   E
@  ( "E
   ( Apj( j" (Aj"6 E
  )!  )!	A ÝC"A6  7   	|7  6 AÔ£6  Aø£6    )"  ) |"  S7 — E
@@  ( Alj( j" ( (  §"
 A ! AV!  6  6 ( (!  )7   B   E
A$ÝC!  6(  6$  A$j¸" ( Aj"6  E
 ($! A 6$@ E
  JA$ÝC  ‹" (Aj"6 
A$ÝC „" (Aj"6A ! E
  A 6@  AÃ jA 6    )!	  Ç A : #  AÀ j!@@   A#jÃE
A€œ - #"j-  A× G
 Avj      )!  )!A ! A : $    B|"  S"7   A$jÃ   B|7   A/jÃ   7 - $"A
G A
Gq
  (8AG
 AÑ¿A°7
    )" 	  	S7 —A ! E
   ( Apj( j" ("E
   Aj"6 
     ( (  @ E
  (" E
   Aj" 6  
   ( (  @ E
  (" E
   Aj" 6  
   ( (   A0j$   ~~~~# A0k"$  A	6( A¡´6$  )$7   AjÞ! A6  AÑ¿6  )7@@    AjÞ"ƒB Y
 B! A : / A : .  )!       S"   BU" B S""   "B~|"7   A/jÃ   B|"7   A.jÃ   7@@ - /A
G
  - .AÿqA
F
 A : / A : .   7   A/jÃ   7   A.jÃ   7   - /" A
F"  A
FrAq"  - .AÿqA
G  !B   S! A0j$  ~~# A0k"$ @@  ( "E
   ( Apj( j" (Aj"6 E
 A j –!  )! Aj  Í@@@ - AG
  ("E
  (
    )"   S7A ! Aj—! Aj  Í@@@ - AG
  ("E
  (
    )"   S7A ! Aj—!	 A(j  Í  ((6 AjAÔ¿‰!
 (! A 6@ E
  ^@ 

     )"   S7A !@    Ó"E
   	6  6@  ( " E
     ( Apj( j" (Aj"6 E
A !@ E
   - Aq
   - Aq
   (Aj"6 ! E
    ( Apj( j" ("E
   Aj"6@ 
     ( (   E
  (" E
   Aj" 6  
   ( (   (!  A 6  E
   ^ (!  A 6@  E
   ^ — A0j$   “
~~~~~~# A k"$  ) "§!  )"!@@ B ˆ"§"AH
 A ! !@@   AjÃ
 B!	@ -   j-  G
  Aj" F
    )"	 B|" 	 S"7A !  B!	  }B S
  AH!
@  )!	  )!  7  7    } 	 AjAß!  )!@ E
   }!	 !@ 

 A !@@   AjÃ
 B!	@ -   j-  G
  Aj" F
    )"	 B|" 	 S"7A !  B!	  }BU
    7 A j$  	Ð~# Ak"$ @ ("E
 A€œ ( "-  j-  !@@A€œ  jAj-  j-  "AÄ F
  A× F
   ¬|" U
   )!   7   AjÃ!   7 E
 A !@A€œ - j-  "A²j     E
  AÄ F
@ BS
 @ A¼j                     )!   B|7   AjÃ!   7 E
 A !@A€œ - j-  "A²j     E
  AÄ F
A! Aj$   /A !@  ÇE
   AÀ j"  (8jA :   —! ˜
~~# A k"$ @@ ("E
  A~j! ( " Aj"j! ) !	  )!
 !@@ P
  
  ) }W
A !   
 AjÅE
@  O
 @ - Aÿq"  j-  G
 @ Aj"A H
  
B|!
  	7  	7   
  AjA ßE
    
7A! 
B|"
BW
    -  F! A ! A j$     )}~~# Ak"$ A !@  5|" B…ƒB S
    )U
   (!  ) "7 ( (!  )!  7     |  ! Aj$  X  (!  A 6@@ E
   ( Apj( j"("E
  Aj"6 
   ( (     ]  (!  A 6@@ E
   ( Apj( j"("E
  Aj"6 
   ( (    A âC f    ( Atj( j"(!  A 6@@  E
     ( Apj( j" ("E
   Aj"6 
     ( (    k    ( Atj( j"(!  A 6@@  E
     ( Apj( j" ("E
   Aj"6 
     ( (   A âC ñ# A€k"$   B	ÑA !@@  A AÝ"E
 @  ( (, "E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   E
  A
6l A¹â6h  )h7PA !@  AÐ jˆE
  A6| AÁÿ6x  )x7H  AÈ jˆE
  A6t AÁÿ6p  )p7@  AÀ j„"E
 A !@ AjîE
  ùA J! ("E
  Aj"6@@ 
   ( (   
A ! 
 A ! A6| A¼þ6x  )x78@  A8jˆE
  A6t A¼þ6p  )p70@  A0j„"
 A !A !@ AjîE
  ùAJ! ("E
  Aj"6@ 
   ( (   
A ! 
 A ! A6| A—ú6x  )x7(@@@  A(jˆE
  A6t A—ú6p  )p7   A j„"E
 A !@ AjîE
  ùA J! ("E
  Aj"6 
  ( (   
A ! 
 A ! A6| Aâþ6x  )x7@@@  AjˆE
  A6t Aâþ6p  )p7  Aj„"E
 A !@ AjîE
  ùA J! ("E
  Aj"6 
  ( (   
A ! 
 A ! A6d AÐ„6`  )`7A !  AjéE
  A6\ AÂþ6X  )X7   AAêE
  Aø j  Í Aø jAÑ¿‰! (x!A ! A 6x@ E
  ^ E
 @AÈ ÝC   )ë"  ÄìE
  ! AÈ âC ("E
  Aj"6 
   ( (   A€j$   £~# A k"$   ) "7  7A !@@   AjˆE
   7   7   „" E
 A !@  AjîE
   ùA J!  ("E
   Aj"6 
     ( (   A j$   ¸~# A k"$   ) "7  7@@@   Ajˆ
  As!   7   7@   „"
 A ! A ! @ AjîE
  ù" AJ   Oq!  ("E
  Aj"6 
   ( (   A j$    Ê# Að k"$  A6l AÁÿ6h  )h70    A0jö¬7  A6d A¼þ6`  )`7(    A(jö6 A6\ A—ú6X  )X7     A jö¬7 A6T Aâþ6P  )P7    Ajö6 A6L AÐ„6H  )H7    Ajö¬7  A6D AÂþ6@  )@7  Ajö!  A 6@  B 78   70   6( A6< A´ƒ68  )87 @@  ÿ"E
 @@ ( (kAuA~j     A ª"A  A J­78 Aª"A H
    6@ ("E
  Aj"6 
   ( (   Að j$    hA !@  )  R
   ("AþÿÿÿK
    (O
   ) Y
   )  Y
   ((AÿÿÿK
   )0 Y
   )8 S!    +A !@  (AI
   )8BS
   (@A G! ÐAAÄW"  6 Aj!A!@@  Atj  Av  sAå’žàl j" 6   Aj"Atj  Av  sAå’žàl j" 6   Aj"Atj  Av  sAå’žàl j" 6  Aj"AÐF
  Atj  Av  sAå’žàl j" 6  Aj!   AÐ6  ˜  Aj!@@  ( "AÏK
  Aj!  Atj( ! ( !A !@ A€€€€xq!  Atj"  Aj"Atj( "AqAtA€¤j(  A j( s  AþÿÿÿqrAvs6  ! AˆG
   A¤j( !Aˆ!@ A€€€€xq!  Atj"  Aj"Atj( "AqAtA€¤j(  Aàsj( s  AþÿÿÿqrAvs6  ! AÏG
 A!  AÀj"  ("AqAtA€¤j(   A j( s Aþÿÿÿq ( A€€€€xqrAvs6    6  Av s"AtA€­±éyq s"AtA€€˜þ~q s"Av s   Jˆ# A k"$ @@A - ÈƒlE
 A (Ìƒl! AjA Ú6 (! (!7!A A: Èƒl   AjAvs AÀ„=lssAs!A  Aj"6ÌƒlAAÄW" 6 Aj!A!@@  Atj Av sAå’žàl j"6   Aj"Atj Av sAå’žàl j"6   Aj"Atj Av sAå’žàl j"6  Aj"AÐF
  Atj Av sAå’žàl j"6  Aj!   AÐ6 @  ("E
   ( !A !@  Atj ð6  Aj" G
  J A j$ )   A :   A¤6   A 6  AjA AÄ Ø6  „  (,!  A 6,@ E
  ½A,âC  ((!  A 6(@@ E
  ("E
  Aj"6 
   ( (    ($!  A 6$@ E
  ^   Š  (,!  A 6,@ E
  ½A,âC  ((!  A 6(@@ E
  ("E
  Aj"6 
   ( (    ($!  A 6$@ E
  ^  AÐ âC Ë# Ak"$ @@ E
   A §6  A$j Aj„ (! A 6 E
 ^  A$jiA !@@   ÷E
 @  (E
 @@ ( "E
  (E
    AøE
   A: A !   A øE
  ("A!O
A,ÝC!  (!  6   A0j6  )7    ¼!  (,!   6, E
  ½A,âCA!@ E
  (" E
   Aj" 6  
   ( (   Aj$   Ã~~# Aà k"$ @@ E
   (Aj"6 E
  ((!   6(@ E
  ("E
  Aj"6 
   ( (   A6\ A‹÷6X  )X7(    A(jö6 A6T A þ6P  )P7     A jö6 A6L A¼þ6H  )H7    AjA÷6@@  (AJ
  A 6D  AÄ j  Aj  Ajù! (D! A 6D E
 ^ A6@ Aÿƒ6<  )<7  Ajñ! A68 Aúƒ64  )47   Ajñ"6DB !@@ 
 A !B !A  Aj 5"P!@ E
  ("­B † AjA  ­„!@@  B ˆQ
 A !A !  § §°7
   AÄ j  Aj  Ajù! (D! A 6D@ E
  ^ E
  ^ Aà j$   ø# A0k"$ @@    úE
 A!  A6 A ! ( "E
  ("E
  Aj" j! !@@ ,  A L
 Aj" G
 A !@  (AH
   6$  6   ) 7  Aj¤6(  A(jš"6, ((! A 6(@ E
  e@   A,j ú"E
   A6  E
 ^  6  6  )7  Aj¦6(  A(j˜"6, ((! A 6(@ E
  e@   A,j ú"E
   A6  E
  ^ A0j$  Ä# Ak"$  A6Œ A‹÷6ˆ  )ˆ7@   AÀ jö! A6 A ! A 6 @@@@@ AH
  A6„ A·„6€  )€70   A0jû"E
@@ A½€‰E
 A ! A 6 A! @@ ( "
  B 7x (! B 7x E
   6|  Aj6x  )x7(@  A(jû"
 A !A ! @@ AG
  A6t A÷Á6p  )p7  AjA ÷"
 A6l A÷Á6h  )h7   AjA€÷! A6d A÷Á6`  )`7    A jA€÷!A !@ A H
  A ÿ6T A6X  )T7   Ajñ6\ A(I!  At!@@ AÜ jAÌ‰
  AÜ jA¥‰E
 A6     ! (\!  A 6\@  E
   ^ Av! ("	E
 AJ!   	Aj"6 
   ( (   ("E
  Aj"6@ 
   ( (    
A! AH
 A6P A÷Á6L  )L78   A8jA(÷Am! A M
 A !@@@@ (   A F
A ! A{jAI
A ! A F
  A7qAF
 A !  6 A! Aj$   ‘# Ak"$ @@  (AH
     ý!@ E
     ‚"6A!@   AjA ƒ
    AjAƒ! E
 ^A!   A ƒ
    Aƒ! Aj$  # A k"$ A  ("  -   !@  ((" E
  A6 Aµ¥6  )7    Ajñ6 AjA’Ý‰! (!  A 6@  E
   ^ A¼qAÀar  ! A j$  Ž~~# Aà k"$ @@ E
   (Aj"6 E
  ((!   6(@ E
  ("E
  Aj"6 
   ( (   A6\ A‹÷6X  )X7     A jö6 A6T A þ6P  )P7    Ajö6 A6L A¼þ6H  )H7    AjA÷6 A 6D A 6@@@  (AH
  A68 Aÿƒ64  )47   Ajñ6< AÀ j A<j„ (<! A 6<@ E
  ^ A60 Aúƒ6,  ),7    ñ6< AÄ j A<j„ (<! A 6<@ E
  ^B !@@ (@"
 A !B !A  Aj 5"P!@ (D"E
  ("	­B † AjA  	­„!A !  B ˆR
  § §°7
A !  AÄ j  ùE
    ( 6   ( 6A! (@! A 6@@ E
  ^ (D! A 6D@ E
  ^ Aà j$   å	# Ak"$   ((! A6„ AÂþ6€  )€7˜@@@  A˜jñ"
 A !A !@ (A0I
   ((! A6ü A“÷6ø  )ø7  Ajñ"E
 A !@ ("A0I
 A !A !	 !
@ E
  Aj! (!A!	 !
@@  (AH
  AM
 AxqA F
  	: Ì  6È  )È7p  
A,j Að j AÐjþ Aj%@@ ( "
 A !A ! Aj! (!  6Ä  6À  )À7ˆ Aj Aˆj% AM
 AxqA F
 A6¼  
A,j6¸  )¸7€ Aj A€j%@ 	E
  A06´  6°  )°7x Aj Aø j% Aj AÐj’%A ! AÐj 
AjA °7
 @@  (AH
  AxqA(F
  	: ¬  6¨  )¨7P  
A4j AÐ j AÐjþ Aj%@@ ( "
 A !A ! Aj! (!  6¤  6   ) 7h Aj Aè j% AxqA(F
 A6œ  
A4j6˜  )˜7` Aj Aà j%@ 	E
  A06”  6  )7X Aj AØ j% Aj AÐj’%  ((! A6Œ AÆ„AÃ„ 6ˆ  )ˆ7H  AÈ jñ"E
 A !@ (A I
 A ! AjA AôØ6 A 6Œ  AÐj6ˆ  )ˆ7@ Aj AÀ j‡% B 7ø B 7ð Aj Aðjˆ% A 6ì   A0j"6è (AM
 A 6ä  Aj6à  )è78  )à70 Aj A8j A0j‰% A 6Ü  6Ø  )Ø7( Aj A(j‡% Aj Aðjˆ%  ((! A6Ô A²–6Ð  )Ð7   A jñ"E
 @@ ("
 A ! B 7È B 7À AÀj Aj A AI×6 A6¬ A6¤  A°j6¨  )¨7  AÀj6   ) 7 Aj Aj Aj‰%A ! - ¹Aá G
  - ºAÿqAä G
  - »AÿqAâ G
  (°  (G
 A! - ¸AÆ F
   ((!  A6Œ Aæð6ˆ  )ˆ7   AjAõ! ^ ^ ^ ^ Aj$   —
~~~# Ak"$  AÈj%@@  ( "
 A !A ! Aj! (!  6Ä  6À  )À7H AÈj AÈ j% A6¼  6¸  )¸7@ AÈj AÀ j%@ - AG
  ( ! A06´  6°  )°78 AÈj A8j% AÈj Aj’% AœjA AôØ6A ! Aj!A !A !	A !
A !A !A !@@@ !A !@  ( "
E
  
(!@@  jAt"A€j  - " 
 	k"M
 @  
k  k"I
  
A  Ø6 j!
 AL
  	k"
At"   KAÿÿÿÿ 
AÿÿÿÿI"AV" j"
A  Ø6!@@ 
 	G
  !A ! 
!@ Aq"E
 @ 
Aj"
 Aj"-  :   Aj" G
  ! 	 
kA|K
 @ 
Aj Aj-  :   
A~j A~j-  :   
A}j A}j-  :   
A|j"
 A|j"-  :    	G
  !  j!  j!
@ 	
  !	 	J !	 	 j 
  I!
@@  ( "
 A !
A ! Aj!
 (!  K
 Aj! 
 	k!A ! A 6˜ B 7  j! 
 j!@ Aj (” 
  ÿ Aj (”   €@ - AG
  Aj (” ( " A0jA0ÿ Aj"AÀ G
  (” (k G
 A6Œ  6ˆ  )ˆ70 Aœj A0j‡% Aœj ˆ%  	6€  ("6x  (” k6|  6„  )€7(  )x7  Aœj A(j A jŠ% AM
@ A|q"AK
 A tA"q
@@@@ 	A
j1  B† 	Aj1  „ 	A	j1  B†„ 	Aj1  B†„ 	Aj1  B† 	Aj1  „ 	Aj1  B†„ 	Aj1  B†„ 	(  "At A€þqAtr AvA€þq AvrrAp­B †„B‚B †„B‚B † 	A
j1  B† 	Aj1  B†„ 	Aj1  B†„ 	Aj1  „„B‚§   6h  	6d  )d7 Aì j Aj“%@ E
  J (p! (l!A !  6`  	6\  )\7 Aì j Aj–%@ E
  J (p! (l!A0!  6X  	6T  )T7 Aì j Aj˜%@ E
  J (p! (l!AÀ !  k"AM
 ApqAF
@ ("E
   6” J ! Aj"AÀ H
  ! Aaj 
Aj-  I
  Aj)  ! Aj)  ! )  ! Aj Aj)  7   Aj 7   Aj 7    7   J 	J Aj$ Å6 ÷	@@ AH
 @   ("  ("kJ
 @@   k"J
   j! !	@@  j" G
  !	 ! !	@ 	 -  :   	Aj!	 Aj" G
    	6 AH
  j!
 	!@ 	 k" O
 @@  j" 	kAq"
  	!A ! 	!@  -  :   Aj! Aj! Aj" G
  	 kAyO
 @  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
    6@ 	 
F
  	 	 
k"k  ×6  F
    k×6   ( "	k j"AL
  	k!A !A !@  	k"At"	  	 KAÿÿÿÿ AÿÿÿÿI"
E
  
AV!  j!@@ Aq"	
  ! !@  -  :   Aj! Aj! Aj" 	G
   j!@ AI
 @  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
  !@   ( "F
 @@  kAq"
  ! !A !	 ! !@ Aj" Aj"-  :   	Aj"	 G
   kA|K
 @ Aj Aj-  :   A~j A~j-  :   A}j A}j-  :   A|j" A|j"-  :    G
   
j!@   ("F
 @@  kAq"
  !A !	 !@  -  :   Aj! Aj! 	Aj"	 G
   kAxK
 @  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
    6   6  ( !   6 @ E
  J ! Å6 ÷	@@ AH
 @   ("  ("kJ
 @@   k"J
   j! !	@@  j" G
  !	 ! !	@ 	 -  :   	Aj!	 Aj" G
    	6 AH
  j!
 	!@ 	 k" O
 @@  j" 	kAq"
  	!A ! 	!@  -  :   Aj! Aj! Aj" G
  	 kAyO
 @  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
    6@ 	 
F
  	 	 
k"k  ×6  F
    k×6   ( "	k j"AL
  	k!A !A !@  	k"At"	  	 KAÿÿÿÿ AÿÿÿÿI"
E
  
AV!  j!@@ Aq"	
  ! !@  -  :   Aj! Aj! Aj" 	G
   j!@ AI
 @  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
  !@   ( "F
 @@  kAq"
  ! !A !	 ! !@ Aj" Aj"-  :   	Aj"	 G
   kA|K
 @ Aj Aj-  :   A~j A~j-  :   A}j A}j-  :   A|j" A|j"-  :    G
   
j!@   ("F
 @@  kAq"
  !A !	 !@  -  :   Aj! Aj! 	Aj"	 G
   kAxK
 @  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
    6   6  ( !   6 @ E
  J ! Å6 B# Ak"$   ((!  A6 Aæð6  )7    Aõ!  Aj$   ß	# A k"$   ((! A6˜ AÂþ6”  )”7@@@@  AÀ jñ"E
  (AK
 A 6œA !A !@@ ( "E
  ("E
  Aôj Aj A  A I"Ö6 AK
A  k! Aôj jA˜¤ Ö6 A 6à  Aôj6Ü  )Ü78 A8j Aäj†%@  (AH
 A !@ A6Ø  Aäj6Ô  )Ô70 A0j Aäj†% Aj"A2G
  A°jAjB 7  A°jAjB 7  A°jAjB 7  B 7° AjAjB 7  AjAjB 7  AjAjB 7  B 7@  ("E
  A°j Aäj A AIÖ6@@@ ("A!O
 @ E
  Aj Aj ×6@@  (AF
 A! Aø j! Að j! Aè j!	@ ! B 7  B 7  	B 7  B 7`@@  ("

  A 6\  Aj6X 
Aq!A !A !@ 
AI
  
A|q!A !A !@ Aà j j A°j j-   s:   Aà j Ar"
j A°j 
j-   s:   Aà j Ar"
j A°j 
j-   s:   Aà j Ar"
j A°j 
j-   s:   Aj! Aj" G
 @ E
 @ Aà j j A°j j-   s:   Aj! Aj" G
  A 6\  Aj6X 
A!O
  
6T  )X7(  Aà j6P  )P7  Aj! A(j A j% 
   A 6Œ  Aj6ˆ A!O
  6„  )ˆ7  A°j6€  )€7 Aj Aj%A !@@@ A—¤j-   Aj Aj"j-  F
  ! A–¤j-   Aj A~j"j-  G
 E
 ! A•¤j-   Aj A}j"j-  F
   6L  Aj6H  B 7H  )H7 Aœj Ajý@ E
  ^ (œ! A j$  ‡
# AÀk"$ @  ("A!O
   ((!  6¼   A0j"6¸  )¸7@   AÀ j   A$j"„A !@  (("E
  A6´ A“÷6°  )°78  A8jñ"E
 A !@ ("AI
 @@  (AG
  A¨jA ) °¤7  A jA ) ¨¤7  A 6Œ A )  ¤7˜ A ) ˜¤7  Aj6ˆ  ("A!O
  6„  6€  )ˆ7  )€7 Aj Aj% (AM
 Aj AjA°7! AàjAjB 7  AàjAjB 7  AàjAjB 7  B 7à AÀjAjB 7  AÀjAjB 7  AÀjAjB 7  B 7À Aàj Aj A  A I×6A!@ !@@  ("
  A 6¼  Aàj6¸ Aq!	A !A !@ AI
  A|q!
A !A !@ AÀj j  j-   s:   AÀj Ar"j  j-   s:   AÀj Ar"j  j-   s:   AÀj Ar"j  j-   s:   Aj! Aj" 
G
 @ 	E
 @ AÀj j  j-   s:   Aj! Aj" 	G
  A 6¼  Aàj6¸ A!O
  6´  )¸70  AÀj6°  )°7( Aj! A0j A(j% 
  AØ j‚% A 6T A˜¤6P  )P7  AØ j A jƒ%@ ( "E
  ("E
   6L  Aj6H  )H7 AØ j Ajƒ% AØ j Aj…% Aàj AjA°7! E! ^ AÀj$   ˜# AÀk"$  ( !@ ("E
  A  Ø6A !A !@@ ( "E
  ("	E
  A j Aj 	A  	A I"Ö6 	AK
A  k! A j jA˜¤ Ö6 AÈj‚% A 6Ä  A j6À  )À7P AÈj AÐ jƒ% A6¼ AÂþ6¸  )¸7H@@   AÈ jñ"
 A !A ! Aj! (!  6´  6°  )°7@ AÈj AÀ jƒ% A6¨ A¼þ6¤  )¤78   A8jö! A6   6¬  A¬j6œ  )œ70 AÈj A0jƒ%@ ( "E
  ("E
   6˜  Aj6”  )”7( AÈj A(jƒ% A6 A þ6Œ  )Œ7    A jö!@@@ 
  AH
  A6ˆ Aæð6„  )„7@   AjAõ
  A6€ A¸¤6|  )|7 AÈj Ajƒ% A AI! AÈj Aà j…% A AI! AÈj Aà j…% AH
A ! @  6\  Aà j6X  )X7 Aj Aà j†%  Aj" A2G
 @ E
   Aà j ×6@ E
  ^ AÀj$ à~# AÀ k"$ @@@@@  ( Aj   ) "7  7   Aj¤6, A,jš!  (,! A 6, E
 e  ) "7  7  Aj¦6, A,j˜!  (,! A 6, E
 e   ) "7   70 A<j ý( !  AÀ j$   	# Ak"$  A 6Œ A 6ˆ@@    AŒj AˆjüE
 @  (AH
  A6„  AÐj6€  )€7 Ajò A°j% A6¬  AÐj6¨  )¨7  A°j % A°j  A0j’%    ‡   ˆ A 6¤@ E
   A §6° A¤j A°j„ (°! A 6° E
  ^ (ˆ"A!O
  ((!  6    A0j"6œ  )œ7p   Að jA  A¤j„@@  (AJ
  AÈjA ) °¤7  AÀjA ) ¨¤7  A 6˜  6  6Œ A )  ¤7¸ A ) ˜¤7°  A°j6”  )”7(  )Œ7  A(j A j% AÐjA“÷ü! A 6ˆ  A°j6„  )„7@   A j Ajý"Ù"E
  ("E
  Aj"6 
   ( (   ( ! A 6 @ E
  ^ ( ! A 6  E
 ^ A°j‚% A 6€ A˜¤6ü  )ü7h A°j Aè jƒ%@ (¤"E
  ("E
   6ø  Aj6ô  )ô7` A°j Aà jƒ% A°j AÐj…% A6Ì  6À  AÐj6È  )È7X  6Ä  )À7P A<q!	 Aq!
 AØ j AÐ j% AI!A!@ !@ E
 A !A !A !@ 
 @ A j j  j-   s:   A j Ar"j  j-   s:   A j Ar"j  j-   s:   A j Ar"j  j-   s:   Aj! Aj" 	G
  
E
 @ A j j  j-   s:   Aj! Aj" 
G
  A6œ  6”  AÐj6˜  )˜7H  A j6  )7@ Aj! AÈ j AÀ j% AÿqAI
   AÐj6ˆ A6Œ  )ˆ78 A8j AÐjAj†% A„jA“÷ü! A 6|  AÐj6x  )x70@   A€j A0jý"Ù"E
  ("E
  Aj"6 
   ( (   ( ! A 6 @ E
  ^ ( ! A 6  E
  ^  ("A!O
A,ÝC!  (!  6´  6°  )°7   Aj¼!  (,!   6,@ E
  ½A,âC (¤! A 6¤ E
  ^ Aj$  â# Aðk"$  Aj‹% A 6Œ   A0j"6ˆ  )ˆ7h Aj Aè jŒ% A6„ A¿ª6€  )€7` Aj Aà jŒ% Aj Aàj%@@  (AH
  A : Ü A : Ø  )Ø7H  Aàj AÈ j Aàjþ A˜j%@@ ( "
 A !A ! Aj! (!  6Ô  6Ð  )Ð7X A˜j AØ j% A6Ì  Aàj6È  )È7P A˜j AÐ j% A˜j Aàj’% Aˆj )è7   )à7€ A´jA“÷ü! A06Ä  Aàj6À  )À7@@@   Aj AÀ jý"Ù"E
  ("E
  Aj"6 
   ( (   ( ! A 6 @ E
  ^ ( ! A 6 @ E
  ^@@  (AH
  A : ¼ A : ¸  )¸7(  AàjAr A(j Aàjþ A˜j%@@ ( " 
 A !A !   Aj!  (!    6´  6°  )°78 A˜j A8j% A6¬  AàjAr6¨  )¨70 A˜j A0j% A˜j Aàj’% A´jA AôØ6 A 6°  Aàj6¬  )¬7  A´j A j‡% B 7˜ B 7 A´j Ajˆ% A06Œ A 6„  6€  Aàj6ˆ  )ˆ7  )€7 A´j Aj AjŠ% Aü jAÃ„ü!  A 6t  Aàj6p  )p7@    Aø j Ajý"Ù"E
  ("E
  Aj"6 
   ( (   ( ! A 6 @ E
  ^  ( !  A 6 @ E
  ^ Aðj$  —# Ak"$   (! A6ü Aæð6ø A6„  : €  Av: ƒ  Av: ‚  Av:   )ø7(  A(jAõ! A6ð Aâ : ‹ AáÈ; ‰ AÔ AÆ  : ˆ  Aôj6ì  )ì7  A jò  (ô6Œ Aø jA AôØ6 A 6t   A0j6p  )p7 Aø j Aj‡% B 7h B 7` Aø j Aà jˆ% A6L A6D  AÐ j6H  )H7  A€j6@  )@7 Aø j Aj AjŠ% A<jA²–ü! A64  AÐ j60  )07 @@   A8j ý" Ù"E
  ("E
  Aj"6 
   ( (    ( !  A 6 @ E
  ^ ( !  A 6 @  E
   ^ Aj$  À# A0k"$ @@@  
 A !  à! A6, AÞ²6(  )(7A !@  AjÒE
  A6$ Aâþ6   ) 7  Aj„"E
 A !@ AjîE
 A ! ùA H
  ùAÿÿÿJ
  A6 AÔˆ6  )7   „"E
 A !@ AjîE
  ùAJ! ("E
  Aj"6 
   ( (   ("E
  Aj"6 
   ( (  @ E
  ("E
  Aj"6 
   ( (  @ E
 AÝC  Š!  ("E
   Aj"6A ! 
     ( (   A0j$   ‡# Ak"$ A$ÝC!@@ E
   (Aj"6 E
  Ÿ" (Aj"6 E
   A 6   6  à! A6 AÔˆ6  )7   ö! ("E
   Aj"6@ 
   ( (    A 6  B 7   6   ‹ ("E
   Aj"6@ 
   ( (   Aj$    ò~# Aðk"$   ( ¤ Aèj  ( «AÝC!  )è"7   7  Aj±" (Aj"6@@@ E
   (!   6@@ E
   ( Apj( j"("E
  Aj"6@ 
   ( (    ("
 A !  ( Apj( j" (Aj"6 E
 A j ¿! à! A6 Aâþ6  )7  Ajö! ("E
   Aj"6@ 
   ( (  @ AH
 @ )  (" ( Alj( j" ( (  Y
 à! à!	@ E
 @@  ("  ("
O
   	6  6  Aj!   ("kAu"Aj"A€€€€O
@@ 
 k"Au"
  
 KAÿÿÿÿ AøÿÿÿI"
 A !
 A€€€€O
 AtÝC!
 
 Atj" 	6  6  
 Atj!	 Aj!@  F
 @ Axj" Axj") 7   G
   (!
  (!   	6   6   6 E
   
 kâC   6 AJ! Aj! 
  Á Aðj$  Å6 ð ¶@  ("E
    6   ( kâC  (!  A 6@@ E
   ( Apj( j"("E
  Aj"6 
   ( (    ( !  A 6 @ E
  ("E
  Aj"6 
   ( (     RA !@   (  ("kAuO
   Atj"(  G
     (Ž" E
    6  ! ¶~~# AÐk"$ A !@@  4" ­|" B…ƒB S
    (" ( Alj( j" ( (  Y
 @  (" E
     ( Apj( j" (Aj"6 E
 Aj  ¿"  Ñ   Ò!  Á AÐj$   Û  B 7  A :   A ;   6  B 7 AÝCŒ!  B 74   6  B 7   A(jB 7   B 7@    A4j60  A 6H    AÀ j6<@ 
 AÝC"B 7  AjB 7  AjB 7  ˆ"AÄ¤6   (!   6@ E
   ( (    (!   6  	   A ƒ  (H!  A 6H@@ E
  ("E
  Aj"6 
   ( (    A<j  (@É  A0j  (4’  (,!  A 6,@ E
  íAÈ âC  ((!  A 6(@ E
  ^  (!  A 6@ E
  ŽAâC  (!  B 7@ E
   ( (    ( !  A 6 @ E
  ÁAÈâC   E @ E
    ( ’   (’ (!  A 6@  E
   ŒAâC AâC^@  (" (
 A @@  ("
   Aj!@  (" ( F!  ! 
  @ " ("
   (d@  (" (
  A M@@  ("
   Aj!@  (" ( F!  ! 
  @ " ("
    (M-~B !@  ( “" E
   -  AG
   )!  @  ( “" 
 A  -  Eç~~# A k"$   6@@@ E
   ( Apj( j" (Aj"6 E
 Aj AjÎ (! A 6@ E
   ( Apj( j"("E
  Aj"6 
   ( (  @@@ - AG
   ( (  ! - E
  )"B	|S
AÈÝC  À!  ( !   6 @ E
  ÁAÈâC  ˜! 
 A !  ( Apj( j"("E
  Aj" 6A !  
   ( (   A j$   t ¹# Ak"$ A !  A 6@  ( B AjÂE
 A !@ , "A H
  Aÿq"’7E
    A
lA |jA  ’76  ( B AjÂE
 A! , "A H
  Aÿq"’7E
    APjA  ’7  (j6 Aj$  \A(ÝC A ˜" ( Apj( j" (Aj"6@ E
 A!@   —E
   A(j ƒ  š!  Í~# Ak"$   A;    ›"7 @@@@ B	S
    œ
@  
 A!  B 7   A: @  
 A!  A:   ž"
 @@  (  Ÿ"E
  à! ("E
  Aj"6@ 
   ( (   E
   (" ( ( ! ("E
  Aj"6@ 
   ( (   
A!  - 
  (H!  A 6H@ E
  ("E
  Aj"6 
   ( (    E
  ž"
  (" ( ( A!   "E
 ("E
  Aj"6 
   ( (  @  ŸAG
   (H!  A 6H@ E
  ("E
  Aj"6 
   ( (  A!  E
  ŸAF
  ž"
A !  (H"E
  
    ! A6 Aíð6  )7   í¡! ("E
  Aj"6@ 
   ( (   E
    (6 (" E
   Aj" 6  
   ( (   Aj$   ÷~# A k"$   ( !  ÄBw|Ñ  ( ! A	6 AØÇ6  )7B !@  AjB€ áE
 @  ( Ð"E
  ^ Aj  ( ÍB ! (!@@ - AG
 @ 
 B !@ (
  A 6B ! Aj˜"B    ( ÄS! (! A 6 E
 ^ A j$  # AÐ k"$   (  ÑA ! A 6H B 7@@@@  A ¢"
   7@   AÀ jA£E
AÝC"B 7   Aj"64  6,  60AÝC" 7   Aj"6(  6   6$   AÀ j¤  ¥"E
  ( A ’  (( ! A6< A«È68  )87@  Ajø"AjA€€€K
   ( •AÝC" 7   Aj"64  6,  60  (( ! A6 Aå²6  )7  Ajø!AÝC" ¬7   Aj"6(  6   6$@@    A,j A j¦
 A !@ (,") "BS
   (  Ñ A 6H B 7@@   AÀ j¢"E
    AÀ j¤@ (@"E
   6D  (H kâCA ! E
  §E
 (,!@ (0 kA	I
 A!@@ (  At"j") BS
 A !   A £E
 (,!@  j) "BS
   (  Ñ A 6H B 7@@   AÀ j¢"E
    AÀ j¤@ (@"E
   6D  (H kâC@ 
 A ! (,! Aj" (0 kAuI
 @ E
 A!  A0j  (4’    A4j60  B 74A!  A: 
@ ( " E
    6$   ((  kâC (," E
    60   (4  kâC AÐ j$  è

~~# A0k"$ AÝCŒ!  ( A€ 6<  ( B Ñ A(j  ( Í@@ (("
 A !A !A !A !A !@@@@ (
  A 6( ^@@@@ - ,AG
  Aj—!  ( )!@@ (("
 B !	 5!	  	}!@@  O
   7  6  Aj!  kAu"
Aj"A€€€€O
@@  k"Au"   KAÿÿÿÿ  AðÿÿÿI"
 A ! A€€€€O
 AtÝC!  
Atj" 7  6  At! !@  F
 @ Apj" Apj") 7  Aj Aj) 7   G
   j! Aj!@ E
   âC !@  kA!O
  ! ! Aj" F
@  ( 6   )7 Aj! Aj" G
  @ A(jAñš‰E
   ( Ê"E
 ^@ A(jAþ‡‰E
  Aj  ( Ë ("E
  6  J@ A(jAÄ¥‰E
   ( A Ò"E
  ( (@ ! (!@@ E
  á!@  ( (, "E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (  A ! AÝC  ‡! E
 ("E
  Aj"6 
  ( (   A(jAÔ¿‰E
   kA G
  Aj( ! ( !
  (  )"Ñ@@  ( A A Ý"
 A !@  ( (@ "E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (  @ 
 A ! à! A6 AìÏ6  )7    ô6 AjAûÇ‰! (! A 6@ E
  ^ ("E
  Aj"6@ 
   ( (   E
 @@ à"è"
 A !@  ( (, "E
   (Aj"
6 
E
 ("
E
  
Aj"
6 

   ( (   (! AÝC  ‡! ("E
  Aj"6 
   ( (  @ 
AÿÿÿK
   
 AÿÿqA   ‰"E
A !@ (" ("F
 @@  Atj( "AÿÿÿK
    
  (! (! Aj"  kAuI
  ŒAâC E
  ("E
  Aj"6 
   ( (   ! Aj  ( Í A(j Aj„  - : , (! A 6@ E
  ^ (("
Å6 ð   (!  A 6  ‡!  (!   6@ E
  ŽAâC  ( A€6<@@  ("( 
 A ! (A G!@ E
    kâC A0j$  ©# A k"$   (H!  A 6H@@ E
  ("E
  Aj"6 
   ( (  @@  (( 
 A! @  ¨"
 A !  A6 Aµ¥6  )7   Ajô6 AjA’Ý‰! (! A 6@ E
  ^@@ 
 A! AÐ ÝCó" (Aj"6 E
@@  (( "
 A ! A6 Aé„6  )7   ÿ!@  (("E
   ( Aj6   6    Ajö! (! A 6@ E
  ^@@ E
   (H!   6H@ 
 A !  (" E
   Aj"6A !  E
 (" E
   Aj"6A!  ! 
  ( (   ("E
  Aj"6 
   ( (   A j$    Ý# Ak"$ A!@@  (( " E
  A6 AÓ‰6  )7    í"E
 @ ñ" E
     (Aj"6 E
 ("E
A!  Aj"6@ 
   ( (    E
   ("E
  (!   Aj"6 
     ( (   Aj$   U@  (  Ÿ" 
 A   à!@  ("E
    Aj"6@ 
     ( (    b@  
 A @@  ñ"E
   (Aj"6 E
  ("E
    Aj"6@ 
     ( (    ¨~# A k"$ @ E
   ( 6   ( Ð6 AjAÝÇ‰! (!A ! A 6@ E
  ^@ E
  B 7A ! A 6 AjA  !@@@  ( ")! Aj Í ("E
A!@ (E
 @ - 
   (  ÑA! Aj—"AÿÿÿK
   ( à!  ( È     ³As! (! A 6@ E
  ^ E
  AG
  
A! ("E
  6  ( kâC@ ( "E
   6  ( kâC  (6   (6  (6A! A j$  Ð~~# Að k"$ A !@@   ) A ¬"E
 @  ( (@ "E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   E
 @@ (
 A ! à! A6l AÐ‡6h  )h70A !@  A0jö"A H
  A6d A«È6`  )`7(A !  A(jö"	A H
   ­7 @@ è"
 A !@  ( (, "E
   (Aj"6 E
 ("E
  Aj"6 
   ( (   (!AÝC  !  (!@@ E
    6@ E
  ŽAâC  (!  	•  A 6  ‡!  (!   6 E
  ŽAâC A6\ Að…6X  )X7 @@@@@@@@@@@  A jÿ"
 A !A !
A !
@ ( (kAO
 A !A !A !A !@@  At"±"E
 @  Ar±"E
  ù! ù!
@ A H
  
AH
 @  O
   
­B † ­„7  Aj!  
kAu"Aj"A€€€€O
@@  
k"Au"   KAÿÿÿÿ AøÿÿÿI"
 A ! A€€€€O
 AtÝC!  Atj" 
­B † ­„7  At!
 !@  
F
 @ Axj" Axj") 7   
G
   
j! Aj!@ 
E
  
 âC !
 ("E
  Aj"6 
   ( (   ("E
  Aj"6 
   ( (   Aj" ( (kAuAvI
  
 G
 
 I
 
! !
 
 k"Au"A AKAÿÿÿÿ AøÿÿÿI"A€€€€O
 At"ÝC"
 	­B †7 @ E
   âC 
 j! 
Aj! E
  	­B †7  Aj! ("E
  Aj"6 
   ( (   A6T Aô6P  )P7@  Ajÿ"
 A ! AÈ j À"( "( E
	A !@@ (" ("G
 A !A !A !A !@ ( " ( ( !@@  O
   6  Aj!  kAu"Aj"
A€€€€O
@@  k"	Au" 
  
KAÿÿÿÿ 	AüÿÿÿI"
 A ! A€€€€O
 AtÝC!  Atj"
 6  At! 
!@  F
 @ A|j" A|j"( 6   G
   j! 
Aj!@ E
   	âC ! Aj" G
  Â ("E
	  Aj"6@ 
   ( (  @  kAu"AO
 A !A !@  G
 A !A!
 !@B !@ 
AqE
  ( "
 j"A   
O"­B † ­„! B ˆ§! §!
 Aj" F
  Å6 ð  
Aq
 A !A$ÝC!  (Aj"6 E
  Ÿ" (Aj"6 E
 ¤ AÈ j «@ 
 F
 A !	 ­! 
!@@ (" 	j"
 I
  
­ ~"B ˆ§
  § (L"
K
  
 	 l"I
  l" 
 kK
 ( "
 j" 
I
  A€€ A€€I!@@  ("(
 A !
@@ ("
  Aj!@  ("
( F! 
! 
  @ "
("
  
(Aj!
 (H!@  
M
   • (!@@ 
 A ! ( "A€€€O
   j!A !
@  
 l"I
   kK
  6<  68  6D   j6@  )87  )@7   Aj Aj ­ 
Aj"
 ("O
 (  
j"AÿÿÿM
   	j!	 Aj" G
  ("E
  Aj"6A! 
   ( (   E
    kâC 
E
  
  
kâC ("E
  Aj"6 
   ( (   ("E
  Aj"6 
   ( (   Að j$   @ ( " ("F
 @@@@@ -   /
"E
  ( (  ‘  ( (  /
 - 	 )  ( (  ( ( Aj" G
 Ä# Ak"$    ( Ð6 AjAÄ¥‰! (!A ! A 6@ E
  ^@@ E
   (   (Ò" E
 @    ( (, "E
   (Aj"6 E
  ("E
   Aj"6 
     ( (   Aj$   ’	~# Aà k"$  B 7H  7P  AÄ jAj"6D AÄ j AÐ j AØ j®  (( ! A64 AÐ‡60  )07   Ajø"¬"78A!@ AH
 @@ ! (H"!	 !
 !@@ E
 @ " 	" ) S"
!  
Atj( "	
 @  F
     
)Y
@@  ")"Y
  !
 ( "
  Y
 ("
  Aj!
AÝC" 6 B 7   7 
 6 @ (D( "E
   6D 
( ! (H È  (LAj6L )8!  7(@@   A(jA £E
  ( ! B 7P   AÐ j¯  (  A8j°  )("78  (  )8Ñ A 6X B 7P@  A ¢E
    AÐ j¤  ¥"E
  (  A8j° ( ! A6$ Aå²6   ) 7   Ajö¬7P   AÐ j¯ A6 AÐ‡6  )7    ø¬78AÝC A !  (!  A 6  ‡!  (!   6@ E
  ŽAâC )8! BY
 A!A ! AÄ j (H± Aà j$  º~~# Ak"$ A!@  ("(" Aj"F
 @@ ) "BS
   ( ")!  Ñ Aj  ( Í  (  ÑA! (!@@@ - AG
 @ 
 A!@ (
  A 6A!AA Aj— (F! (! A 6 E
 ^ AF! !@@ ("E
 @ "( "
  @  ("( G! ! 
   G
  Aj$  ¨# Ak"$ A !@@  (( "E
  A6 A—‰6  )7   í"E
 @@ éE
 @ é"
 A !  (Aj" 6  
A ! ñE
   ( ñ(" E
 @    ( (, "E
   (Aj"6 E
  ("E
   Aj"6 
     ( (   (" E
   Aj" 6  
   ( (   Aj$   F  (H!  A 6H@@ E
  (" E
   Aj" 6  
   ( (   
   (( Q# Ak"$ @@  (( " 
 A !  A6 Aé„6  )7    ÿ!  Aj$   é~  ( ")!  Ñ  (   (AÝ!  (  Ñ@@@ E
  E
  ( G
@  (H"
  @ (,"
  @   (G
  @@ E
   (Aj" 6  
 A   »  »E
   (" E
    Aj" 6@  
   ( (  A  ¯~~@ (E
  ) ! ) "§!@@@ ( ( 
  B ˆ§! §! BÿÿÿÿX
 B ˆ§" §"( "	I
@@ 	E
  	Aq!
A !@@ 	AO
 A ! ! 	Axq!A ! !@ "
Aj! Aj" G
  
Aj(  "At A€þqAtr AvA€þq Avrr!@ 
E
 @ At -  r! Aj! Aj" 
G
   @ B ˆ§  Bÿÿÿÿ/X
  Aj(  	j"I
 Aj( "  kK
@@ 
 A !  j! Aq!A !@@ AO
 A ! Axq!
A !@ "
Aj! Aj" 
G
  
Aj-  At 
Aj-  Atr 
Aj-  rAt"
 
Aj-  r!@ E
 @ At"
 -  r! Aj! Aj" G
  
AÿÿK
  (  Aÿÿq‘ B€€€€pƒB€€€€Q
 Aj( "  	kK
A !A !@ E
   	j! Aq!
A !@@ AO
 A ! Axq!A !@ "
Aj! Aj" G
  
Aj(  "At A€þqAtr AvA€þq Avrr! 
E
 @ At -  r! Aj! Aj" 
G
 @  (" (E
 @@  ("
   Aj!@  ("( F!
 ! 

  @ "("
  (!  K
 Bÿÿÿÿ/X
   	j"I
 Aj( "  kK
@@ 
 A !  j! Aq!A !
@@ AO
 A ! Axq!
A !@ "Aj! Aj" 
G
  Aj(  "At A€þqAtr AvA€þq Avrr! E
 @ At -  r! Aj! 
Aj"
 G
       BÿÿÿÿX
 B€€€€pƒB€€€€Q
  ( "I
 Aj( "	  kK
@@ 	
 B !  j! 	Aq!
A !@@ 	AO
 A ! 	Axq!A !@ "
Aj! Aj" G
  
Aj(  "At A€þqAtr AvA€þq Avrr!@ 
E
 @ At -  r! Aj! Aj" 
G
  ­! Bÿÿÿÿ/X
  	 j"I
 Aj( "  kK
A !@ E
   j! Aq!
A !@@ AO
 A ! Axq!A !@ "
Aj! Aj" G
  
Aj-  At 
Aj-  Atr 
Aj-  rAt" 
Aj-  r!@ 
E
 @ At" -  r! Aj! Aj" 
G
  AÿÿK
  (  AÿÿqA   Ý~~@  F
   Aj!@  (! !@@@   ( F
  ! !@@ E
 @ "("
  @  ("( F! ! 
  ) ) "S
  ! ! E
@@  ")"	Y
  ! ( "
 	 Y
 ("
  Aj! Aj  "( 
   !AÝC! ) !  6 B 7   7  6 @  ( ( "E
    6  ( !  ( È    (Aj6 Aj" G
 Ú# A k"$ @@@@  ("  ("O
 @  G
   ) 7    Aj6 Aj! !@ Axj" O
   ) 7  Aj!   6@  F
    k"k  ×6  ) 7    ( "kAuAj"A€€€€O
   Aj6@@  k"Au"   KAÿÿÿÿ AøÿÿÿI"
 A ! A€€€€O
 AtÝC!  6    kAuAtj"6   Atj6  6 Aj µ ("!@   ( "F
  ! !@ Axj" Axj") 7   G
   6  (" k! (!@  F
    ×6  (! (!  ( !   6   6    j6  6  (!   (6  6  6@  F
     kAjAxqj6@ E
    kâC ! A j$  Å6 ð Õ@  ("  ("O
 @  G
   ) 7    Aj6  Aj! !@ Axj" O
   ) 7  Aj!   6@  F
    k"k  ×6  (!    IAtA   Mj) 7  @@   ( "kAuAj"A€€€€O
 @@  k"Au"   KAÿÿÿÿ AøÿÿÿI"
 A !A ! A€€€€O
 At"ÝC!  j!   k"j!@  G
 @  M
   AuAjA~mAtj!A Au  F"A€€€€O
 At"ÝC"	 j! 	 AtAxqj! E
   âC  ( !  ) 7  !@  F
  ! !@ Axj" Axj") 7   G
  Aj!  (" k!@  F
    ×6    j6  ( !   6   (!   6@ E
    kâC Å6 ð % @ E
    ( ±   (± AâC˜# AÐ k"$   (  Ñ A 6L B 7D@   AÄ j¢"E
    AÄ j¤@ (D"E
   6H  (L kâCA !@@ E
   ¥"E
   (( ! A6@ A«È6<  )<7@  AjøE
   (( ! A68 Aå²64  )47  Ajø! AÝC"6(  Aj"60  7   6, AÝC"6  Aj"6$  ¬7   6 A !AÝC A !  (!  A 6  ‡!  (!   6@ E
  ŽAâC@    A(j Aj¦E
 @ (") BS
 A !   A £E
@ (, (("kA	O
 A!A!@@ ( At"j") BS
 A !   A £E
 ((!@  j) "BS
   (  Ñ A 6L B 7D@   AÄ j¢"E
    AÄ j¤@ (D"E
   6H  (L kâC@ 
 A ! ((!A! Aj" (, kAuI
 @ (" E
    6    ($  kâC ((" E
   6,   (0  kâC (" E
   Aj" 6  
   ( (   AÐ j$   ©~~# A k"$ A!@ E
 @ 
 @  ( ")" ­B~|" B…ƒB Y
 A !  ÑA ! ( ( k"	Am" j"
 I"
 A  
 "A€€K
    ( ÄB§K
 @@  ( ( "kAm"
M
    
k´  
O
    Alj6A AVA A Ø6! !
@@  ( !  6  
A€ 
A€I"Al6  )7@  AjÆ"
  !  
k" j!A !@ (  	j Alj Alj"  j6 @@  Al"j"
Aj-  Aæ G
  B 7A !
@ 
˜"B R
 A ! 
,  "A€q
 APjA	K
 
Aj,  "A€q
 APjA	K
 
Aj,  "A€q
 APjA	K
 
Aj,  "A€q
 APjA	K
 
Aj,  "A€q
 APjA	K
 
Aj,  "A€q
 APjA	K
 
Aj,  "A€q
 APjA	K
 
Aj,  "A€q
 APjA	K
 
Aj,  "A€q
 APjA	K
 
A	j,  "A€q
B ! APjA	K
  7 AöŸ k6  
Aj6  )7   ð;
A!
  
:  Aj" G
  ! 
 k"

  J A j$  ü@  ("  ("kAm I
 @ AlE
  A  Al"Ø6 j!   6@@   ( "kAm" j"A«ÕªÕ O
 @@  kAm"At"   KAªÕªÕ  AÕªÕ*I"
 A ! A«ÕªÕ O
 AlÝC! Al!  Alj"!@ AlE
  A  Al"Ø6 j!  j!@  F
 @ Ahj" Ahj") 7  Aj Aj) 7  Aj Aj) 7   G
   (!  ( !   6   6   6 @ E
    kâCÅ6 ð Ö@@@  ("  (F
  !@  ("  ( "M
   k!   kAuAjA~mAt"j!@  F
    ×6  (!    j"6    j6A  k"Au  F"A€€€€O
 At"ÝC" j!	  AtAxqj"!@  F
    kj! !@  ) 7  Aj! Aj" G
    	6   6   6   6  E
   âC  (!  ) 7     (Aj6ð ›# A k"$   (H!@@  ((" E
     ( Aj6   (! B 7@ E
   6   Aj6  )7  Aj…!  ^ B 7 B 7  Aj…! A j$  
   (({A !@@  (( " E
   è" E
 @    ( (, "E
   (Aj"6 E
  ("E
   Aj"6 
     ( (    Ý# Ak"$ A!@@  (( " E
  A6 A¿«6  )7    í"E
 @ ñ" E
     (Aj"6 E
 ("E
A!  Aj"6@ 
   ( (    E
   ("E
  (!   Aj"6 
     ( (   Aj$   Â~A !A !@  ("(E
 @@ ("
  Aj!@  ("( F! ! 
  @ "("
  (!@  K
   AÀ j!@@@@@  (@"
  ! ! !@   ( I"!  Atj( "
 @  F
   (O
@@  "("O
  ! ( "
  O
 ("
  Aj!AÝC" 6 B 7   6  6  !@  (<( "E
    6< ( !  (@ È    (DAj6DA !@  ( “"E
 @@ -    )"BS
    ¬!   (»"E
    (  (! ("
 !@  ("( G! ! 
   @ "( "
 @  (< G
    6<    (DAj6D  (@ ï AâC €~  AÀ j!@@  (@"E
  !@   ( I"!  Atj( "
   F
 A !  (O
  A4j!@  (4"E
  !@   ( I"!  Atj( "
   F
   (I
  (A !  ( “"E
  - AG
  )"BS
  !@@@@ ( "E
 @@  "("O
  ! ( "
  O
 ("
  Aj!AÝC" 6 B 7   6  6  !@  (<( "E
    6< ( !  (@ È    (DAj6D@@    ¬"
 A !  (Aj"6 E
@  ( (@ "E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   ‰! !@@ ( "E
 @@  "("O
  ! ( "
@  I
  ! ("
  Aj!AÝC"A 6  6  6 B 7   6  !@  (0( "E
    60 ( !  (4 È    (8Aj68 (!  6@ E
  ŒAâC ("E
  Aj"6 
   ( (   ("
 !@  ("( G! ! 
   @ "( "
 @  (< G
    6<    (DAj6D  (@ ï AâC 
   ( Ä @  (," 
 A   ( @  (H" 
 A   û
   ( èÙ~# AÀ k"$   A(j ƒ  B 7   A : 
A!@@   —E
   ( è!  (,!   6,@ E
  íAÈ âC  (,!@ 
   š!  A:    )0"7   7(  (  Ñ A 6< B 74@   A4j¢"E
    A4j¤@ (4"E
   68  (< kâC@@@@@@ 
    A(jA£
  
A!@  ¥"
 A !A !  ( A ’  (( ! A6$ A«È6   ) 7  Ajø"AH
 Aj!  ("(E
 ("
 Aj!@  ("( F! ! 
    B 7   A: @ "("
  (!  F
   
 A!  ž"
 @@  (  Ÿ"E
  à! ("E
  Aj"6@ 
   ( (   E
   (" ( ( ! ("E
  Aj"6@ 
   ( (   
A!  - 
  ©  E
  ž"
  (" ( ( A!   "E
 ("E
  Aj"6 
   ( (  @  ŸAG
   ©A!  E
  ŸAF
  ž"
A !  (H"E
  E
    ! A6 Aíð6  )7  Ají¡! ("E
  Aj"6@ 
   ( (   E
    (6 (" E
   Aj" 6  
   ( (   AÀ j$   Â
~# A k"$   7A !@   AjA £E
  B 7  AjAj"6A !@@@ )"P"
 ! !@@ E
 @@  ")"Y
  ! ( "
  Y
 ("
  Aj!AÝC"	 6 	B 7  	 7  	6 @ (( "E
   6 ( !	 ( 	È  (Aj6   AjA £E
@@ ("E
  )! ! !	@ "
 	" ) S"!  Atj( "	
   F
   
  )Y
 !  F
    A0j  (4’    A4j60  B 74  A: 
 Aj (± A j$  ¿~# Ak"$   (( ! A6 AÐ‡6  )7 A !@@  ö"A N
 A! E
   (!A !  A 6  A0j  (4’    A4j60  B 74@   ¬"²
    Á
   B 7 A!   6 Aj$  ë# A k"$   A 6  B 7  (   6Ä ( B Ñ@@ Aj ( Í@@@ - AG
  Aj ( Í Aj Aj„  - ":  (!  A 6@@  E
   ^ - Aq
A ! Aq
 A ! Aj ( Í Aj Aj„  - :  (!  A 6@  E
   ^A ! AjAÔ¿‰E
@ ( A Ò" E
   ("E
   Aj"6 
     ( (   Aj ( Í Aj Aj„  - :  (!  A 6@  E
   ^ AjAÑ¿‰
@ AjAÄ¥‰E
  ( A Ò" E
  ("E
   Aj"6 
    ( (  @ AjAØÇ‰E
  Aj ( Í (!  A 6  E
  ^@ AjAÝÇ‰
 A !@ Aj ( Í Aj Aj„  - :  (!  A 6@  E
   ^@ (" E
   (E
  AjAØÇ‰E
  ( Í ( !  A 6   E
   ^A! (!  A 6@  E
   ^ 
  ( A 6Ä A j$  ç~# A k"$ A!A€ AVA A€ Ø6!  ( B Ñ@@@ P
 @@ B€  B€ S"§"A O
  ( !  6  6  )7A !  AjÆE
  6  6 ( ( !  )7     E
  Bÿ?ƒ}"B R
 A! E
 J A j$      ‰AâC A›   Ajƒ  ˆ" B 7@  AÜ¤6   B 7$  B 7\  B 7P   6L   6H  A 6t   6p  A 6l  B 7d  A,jB 7   A4jB 7   A<jA ;     AÜ j6X   6  (L  6  Þ  AÜ¤6   (t!  A 6t@ E
   ( (    (t!  A 6t E
   ( (    (p" ( (    A 6p@  (d"E
    6h   (l kâC  AØ j  (\É  (T!  A 6T@ E
   ( (    (P!  A 6P@ E
  †AâC  (L!  A 6L@ E
   ( (    (H!  A 6H@ E
   ( (  @@  (0"E
  !@   (4"F
 @ Axj"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
   (0!   64   (8 kâC  (,!  A 6,@ E
  ("E
  Aj"6 
   ( (    ((!  A 6(@ E
  ("E
  Aj"6 
   ( (    ($!  A 6$@ E
  ‘AÐ âC  ‰"Aj„  % @ E
    ( É   (É AâC
   ÈAø âC @  ($" 
 A    ºÔ    ($“6@@    ($Ÿ"E
  á!  ((!   6( E
  ("E
  Aj"6 
   ( (    Í@@  ((
 A !   (h"  (d" kAL
   G! @ E
  ("E
  Aj"6 
   ( (     ¾# A k"$ @@  ($(,"
 @  Î"  (h  (d"kAu"M
   Aä j  kÏ  O
    Atj6h@@@@   ((""
  A6 AïØ6  )7A  AjÒE
 é! A6 AïØ6  )7  AjÒ! ("E
  Aj"6@ 
   ( (   
@  Î"  (h  (d"kAu"M
   Aä j  kÏ  O
    Atj6h  (!@@ ("  (h  (d"kAu"M
   Aä j"   kÏ  ( !  O
     Atj6h  Atj 6  A j$ # A0k"$ @@@  ((" 
 A ! A6 Aìœ6  )7@   Ajû" 
 A !@@  (Aj     ( (   A6$ AÍ6   ) 7 @   ˆE
    6    (Aj"6 E
 B 7  Aj"6 A(j Aj  Aj AjÓ (! A 6@ E
  ("E
  Aj"6 
   ( (   A(j   AjÔ ((!  - ,! Aj (Õ  A  !  ("E
   Aj"6A! 
     ( (   A0j$   â@  ("  ("kAu I
 @ AÿÿÿÿqE
  A  At"Ø6 j!   6@@   ( "kAu" j"A€€€€O
 @@  k"Au"   KAÿÿÿÿ AüÿÿÿI"
 A ! A€€€€O
 AtÝC! At!  Atj"!@ AÿÿÿÿqE
  A  At"Ø6 j!  j!@  F
 @ A|j" A|j"( 6   G
   (!  ( !   6   6   6 @ E
    kâCÅ6 ð  @  (h  (dkAu" AJ
    f@  ($"
 AÐ ÝC  !  ($!   6$ E
  ‘AÐ âC  ($!@   ™"
     ($- As: < f@  ($"
 AÐ ÝC  !  ($!   6$ E
  ‘AÐ âC  ($!@   À"
     ($- As: < Î# Ak"$ A !@@   Aj Aj ‚"( "
 AÝC" ( "6@ E
   (Aj"6 E
  (6 B 7   6  !@ ( ( "E
   6  ( ! ( ÈA!  (Aj6   :    6  Aj$  ã
# A0k"$  A6, Aú‰6(  )(7@@@  Ajö"AjAýÿ?K
   A:    6  A6$ AÍ6   ) 7@@@@  Aj€"E
  ( (G
A !  A:   A 6  Aj!A !A !@@  «"	E
  !
@@ ( "E
 @ 
  ( 	I"!
  Atj( "
  
 F
  	 
(I
 A! 	 	(Aj"6 E
@@ 	á
  ! !@ ( "
E
 @@ 	 
"("
O
  ! ( "

 
 	O
 ("

  Aj!AÝC" 	6 	 	(Aj"
6 
E
  6 B 7   6  !@ ( ( "
E
   
6  ( ! ( È  (Aj6 Aj 	 Ô@@ - "
   A :   A :   ( j! !	@@ ("
E
 @ 
"( "

  @ 	 	("( G!
 !	 

 @ (  G
   6   (Aj6 ( ï (! A 6@ E
  ("
E
	  
Aj"
6 

   ( (   AâCA !	 
 Aj!@ Aÿÿ?N
 A !  A :   A :  A!@ 	E
  	("
E
 	 
Aj"
6 

  	 	( (      Aj" ( (kAuI
  AjAú‰ü!AÝC ñ"
 
(Aj"	6 	E
@   
þ"
E
  
 
("	Aj"6 E
 
 	6 	
  
 
( (   ( !
 A 6 @ 
E
  
^  A:    6  ("E
  Aj"6 
   ( (   ("E
   Aj"6@ 
   ( (   A0j$  i@@ E
    ( Õ   (Õ (!  A 6@  E
   ("E
   Aj"6 
     ( (   AâC ®
# A0k"$ A !@@ ( A H
   - =
 @  (0 Atj( "E
   (Aj"6 E
 A6, AÍ6(  )(7@@  Aj€"
   (4Axj"( !A ! A 6 @ E
  ("E
  Aj"6 
   ( (     64 ( AG
  (d Atj (6 @@ A€I
   (4Axj"( !A ! A 6 @ E
  ("E
  Aj"6 
   ( (    A: =   64A !@  A0j"	(  At"
jAj( " ( (kAuO
  Aj!A !@ ( E
    µ   «"6$@@ 
   ( Aj6 A! 	(  
jAj" ( Aj6 @  G
 A! 	(  
jAj" ( Aj6  A6  AÍ6  )7@  Ajˆ
   (d  ( kAtjAj ($(6   ( Aj6   (0 
jAj" ( Aj6 A ! ( 
 ($! A 6$@ 
  ! ("E
  Aj"
6A!@ 
E
  !  ( (   !@  (4"  (0kAu G
  A 6@@   (8O
  A 6  ($!
 A 6$ ( !  
6 @ E
  ("
E
  
Aj"
6 

   ( (    (6 Aj! 	 A$j Aj×!   64     Ö!
@  (4  (0"kAu" G
   
jAj" ( Aj6   (4  (0kAu!@@  G
  ( E
   - =AG
A!@ 
  
! ("E
  Aj"6@ E
  
!  ( (   
!A ! 
E
  
("E
 
 Aj"6 
  
 
( (   ($! A 6$@ E
  ("
E
  
Aj"
6 

   ( (  @    Aj" ( (kAuI
  	(  
jAj(  ( (kAuG
   (4Axj"( ! A 6 @ E
  ("E
  Aj"6 
   ( (     64 (" E
   Aj" 6  
   ( (   (" E
   Aj" 6@  
   ( (   ! A0j$   ²@@@  (  ( "kAu"Aj"A€€€€O
 A !@  ( k"Au"   KAÿÿÿÿ AøÿÿÿI"E
  A€€€€O
 AtÝC! ( ! A 6   Atj" 6   ( 6  Atj! Aj!@@  ("  ( "G
  !@ Axj"( ! A 6  Axj" 6  A|j A|j( 6  ! !  G
   (!  ( !   6   6   (!   6@  F
 @ Axj"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
 @ E
    kâC Å6 ð    (d Atj( A G„# A k"$ A !@@ A H
   (h  (d"kAu"AL
  O
 @  Atj( "E
    "E
 @  ( (, "E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   
@  (("
 A ! A6 Aìœ6  )7@  Ajû"
 A !@@ (Aj   ( (    6@  (0"  (4G
   A : =  A 6@  (8! A 6@@  O
  A 6 A 6  6  Aj!  A0j Aj Aj×!   64  Aj"  (@k6    AjA Ö!   6@ (!  A 6  E
   ("E
   Aj"6 
     ( (   A j$   4 @   Ù"E
 @@ (Aj    ( (      (d Atj 6 B@  (P"
 AÝC…!  (P!   6P E
  †AâC  (P! õ@A$ÝC „"("AF
   Aj"6 E
    ’ (!@@@  (\"
   AÜ j"!@@  "("O
  ! ( "
  O
 ("
  Aj!AÝC" 6 B 7   6  6 @  (X( "E
    6X ( !  (\ È    (`Aj6`  r@ 
 A   AÜ j!@@  (\" E
  (! !@     ( I"!   Atj( " 
   F
   (O
 !  G‹# A0k"$ @@@@  (h"  (d"G
 A !  kAu"A AK!A !A !A !@  Atj( " F
     Aq!  Er! Aj" G
   6$@  (("
 A! A6, Aìœ6(  )(7@  Ajû"
 A! A 6 @@  A$j  A jA à"A N
 A!A!  (h  (dkAu"AL
  O
 @@@   "
  A6, AïØ6(  )(7A  AjÒ
 é! A6, AïØ6(  )(7  AjÒ! ("E
  Aj"6@ 
   ( (   E
  (d Atj 6  ! ("E
  Aj"6 
   ( (   A0j$   à# A0k"$  A6, AÍ6(  )(7@@@   Ajˆ
 @   (G
  ( !@ ( "E
   Aj6   ( Aj6 A! A6$ AÍ6   ) 7A!   Ajÿ!@@ AÿJ
  E
  A6 Aú‰6  )7    ö!@@ ( "	 I
   	 k6   (  j6 @  ("	 ("
kAuG
 A !@@  ¥"
E
 @ 
ñ"	E
  	 	(Aj"6 E
 
("E
 
 Aj"6@ 
  
 
( (   	E
 @ 	("
 G
  (  j! 	("E
 	 Aj"6@ 
  	 	( (   
 G
 
 F
 Aj" G
  (!
 (!	 	 
F
  Aj!A !
@@  
¬"	E
 @@ 	  G
 A! 	    à"  AJ"! 	("E
 	 Aj"6@ 
  	 	( (      
Aj"
 ( (kAuI
 A! E
 ("	E
  	Aj"	6 	
   ( (   A0j$   Ð# A0k"$  A6( AìÏ6$  )$7    Ajô6,A !@@ A,jAìœ‰
 A! A,jAïØ‰
  A6  AÍ6  )7    ˆ! AjAìÏü! AìœAïØ 6@     Aj Ajîþ"E
 @ (Aj   ( (   ( ! A 6 @ E
  ^ As! (,! A 6,@ E
  ^  ("E
    Aj"6@ 
     ( (   A0j$    @  ($" 
 A    ¾   (L"    ( (    (L"    ( (    (L"    ( ( ã# Ak"$ @A$ÝC  Aj"â"("AF
   Aj"6 E
    ’  ((!   6(@ E
  ("E
  Aj"6 
   ( (    ((!@  AjAìÏü" AjAÌÄçþ"E
   ("Aj"6 E
  6 
   ( (   ( ! A 6 @ E
  ^A$ÝC â"("AF
   Aj"6 E
    ’@  AjAìÏü" Aj"Aìœèþ"E
   ("Aj"6 E
  6 
   ( (   ( ! A 6 @ E
  ^ AjAú‰ü!AÝCA ñ" (Aj"6 E
 @   þ"E
   ("Aj"6 E
  6 
   ( (   ( ! A 6 @ E
  ^ AjAÍü!A$ÝC –" (Aj"6 E
 @   þ"E
   ("Aj"6 E
  6 
   ( (   ( ! A 6 @ E
  ^  ((! AjAìœü! (!AÝC   Å" (Aj"6 E
 @   þ"E
   ("Aj"6 E
  6 
   ( (   ( ! A 6 @ E
  ^A$ÝC â"("AF
   Aj"6 E
    ’  (,!   6,@ E
  (" E
   Aj" 6  
   ( (   (" E
    Aj" 6@  
   ( (   Aj$  ¤# Ak"$ AÝC!   ( " 6@  E
     ( Aj6   Aj Aj ü"×"   (Aj"6@ E
  ( ! A 6 @ E
  ^ (! A 6@ E
  ë Aj$    ¤# Ak"$ AÝC!   ( " 6@  E
     ( Aj6   Aj Aj ü"×"   (Aj"6@ E
  ( ! A 6 @ E
  ^ (! A 6@ E
  ë Aj$    –# Ak"$ @A$ÝC  Ajâ"("AF
   Aj"6 E
    ’@  AjAìÏü" AjAïØêþ"E
 @ (Aj   ( (   ( ! A 6 @ E
  ^  (Aj"6 E
  (!@@    ëE
  !    ” (" E
   Aj"6A !  
   ( (   Aj$    ¤# Ak"$ AÝC!   ( " 6@  E
     ( Aj6   Aj Aj ü"×"   (Aj"6@ E
  ( ! A 6 @ E
  ^ (! A 6@ E
  ë Aj$    ý	# AÀ k"$ @@@  (("
 A !  (Aj"6 E
 A64 Aìœ60  )07@@  Ajü"
 A !  (h  (dkAu"AL
@  K
 @@  G
  A6, AÍ6(  )(7  Aj! (!AÝC   Å" (Aj"	6 	E
@  ¼"E
   ("	Aj"
6 
E
  	6 	
   ( (   AjAú‰ü!AÝC Ajñ"	 	(Aj"
6 
E
@   	þ"	E
  	 	("
Aj"6 E
 	 
6 

  	 	( (   ( !	 A 6 @ 	E
  	^ AjAüŠü! (!	AÝC   	Å"	 	(Aj"
6 
E
@   	þ"	E
  	 	("
Aj"6 E
 	 
6 

  	 	( (   ( !	 A 6 @ 	E
  	^  A : =  A 6@@  (0"  (4"F
 @ Axj"( !	 A 6 @ 	E
  	("
E
 	 
Aj"
6 

  	 	( (    G
    64 ("E
  Aj"6 
  ( (    6  (Aj"6 E
 B 7   A j"6 A8j Aj  Aj AjÓ (! A 6@ E
  ("	E
  	Aj"	6 	
   ( (  @ E
   (Aj"6 E
     A Ajì! Aj ( ÕA ! E
  Aä j"( !	  (6  	 Atj Ají 
 A! ("E
  M!	  Aj"6@ 
   ( (   	! ("E
  Aj"6 
   ( (  @ E
  ("E
  Aj"6 
   ( (   AÀ j$   Í
	# AÐ k"$  A6L AÍ6H  )H7 @@@  A j€"
 A !@@ ( (G
 A! Aj!	A !
@@  
«"E
   (Aj"6 E
@@@@ áE
 @ E
  Aj!@@ E
  (!AÝC   Å" (Aj"6 E
	@  
 ¹"E
   ("Aj"
6 
E
  6 
   ( (   AÄ jAüŠü! (!AÝC   Å" (Aj"
6 
E
	@   þ"E
   ("
Aj"6 E
  
6 

   ( (   ( ! A 6 @ E
  ^A!  
´A! AÄ jAú‰ü! A6@ Aú‰6<  )<7  Ajö!
AÝC 
 jñ" (Aj"
6 
E
@   þ"E
   ("
Aj"6 E
  
6 

   ( (   ( ! A 6 @ E
  ^  A : =  A 6@@  (0"  (4"F
 @ Axj"( ! A 6 @ E
  ("
E
  
Aj"
6 

   ( (    G
    64A !A!
A ! A68 Aú‰64  )47@   Ajö"H
   k! 	! 	( "!@@ 
  	!@   ( I"
!  
Atj( "
 @  	F
   (I
 A!
A !@@  "("O
  !	 ( "
  O
 ("
  Aj!	AÝC"
 6  (Aj"6 E
 
 6 
B 7  	 
6  
!@ ( ( "E
   6  	( ! ( ÈA!
  (Aj6@ E
   (Aj"6 E
@       ìE
  AÄ jAú‰ü! A60 Aú‰6,  ),7  Ajö!AÝC AA jñ" (Aj"
6 
E
@   þ"E
   ("
Aj"6 E
  
6 

   ( (   ( ! A 6 @ E
  ^A!
@@ 
("
  
!@  ("( G! ! 
  @ "( "
 @ (  
G
   6   (Aj6 ( 
ï 
(! 
A 6@ E
  ("E
  Aj"6 
   ( (   
AâCA!
A!@ E
  ("E
  Aj"6 
   ( (   E
 A! 
Aj"
 ( (kAuO
 
AF! ("E
  Aj"6 
   ( (  @ E
  ("E
  Aj"6 
   ( (   ("E
   Aj"6@ 
   ( (   AÐ j$   Ú# A k"$ @@@@  ("  ("O
 @  G
   ( 6    Aj6 Aj! !@ A|j" O
   ( 6  Aj!   6@  F
    k"k  ×6  ( 6    ( "kAuAj"A€€€€O
   Aj6@@  k"Au"   KAÿÿÿÿ AüÿÿÿI"
 A ! A€€€€O
 AtÝC!  6    kAuAtj"6   Atj6  6 Aj ï ("!@   ( "F
  ! !@ A|j" A|j"( 6   G
   6  (" k! (!@  F
    ×6  (! (!  ( !   6   6    j6  6  (!   (6  6  6@  F
     kAjA|qj6@ E
    kâC ! A j$  Å6 ð §# Ak"$ AÝC!   ( " 6@  E
     ( Aj6   Aj Aj ( ü"×"   (Aj"6@ E
  ( ! A 6 @ E
  ^ (! A 6@ E
  ë Aj$    Ó@@@  ("  (F
  !@  ("  ( "M
   k!   kAuAjA~mAt"j!@  F
    ×6  (!    j"6    j6A  k"Au  F"A€€€€O
 At"ÝC" j!	  A|qj"!@  F
    kj! !@  ( 6  Aj! Aj" G
    	6   6   6   6  E
   âC  (!  ( 6     (Aj6ð Ý@@@  (,"E
   (Aj" 6  
A !  ($"E
  ¹"E
 AÝC   Å" (Aj"6 E
@@ Ö"
 A !@  ( (, "E
   (Aj"6 E
 ("E
  Aj"6 
   ( (    (,!   6,@ E
  ("E
  Aj"6@ 
   ( (    (,!@ E
   (Aj" 6  E
 (" E
   Aj" 6  
   ( (    Í@@@  (,"E
   (Aj" 6  
  ð"
 A$ÝC  Ajâ"("AF
  Aj"6 E
   ’  (,!   6,@ E
  ("E
  Aj"6@ 
   ( (    (,"
 A   (Aj" 6  E
   @  ($" 
 A   «# A0k"$ @@@  (("
 A ! A6 Aìœ6  )7@  Ajû"
 A !@@ (Aj   ( (   A6$ Aú‰6   ) 7   ö!@ A H
   N
    Ù"E
   6  (Aj"6 E
 B 7  Aj"6 A(j Aj  Aj AjÓ (!A ! A 6@ E
  ("E
  Aj"6 
   ( (  @    A A  AjìE
   (h"  (d Atj"Aj"k!@  F
    ×6    j6h (! Aj (Õ ("E
  Aj"6 
  ( (   (" E
   Aj" 6A !  
   ( (   A0j$   Æ@@ E
   (d  (hF
 A !@@   Ù"E
  ("E
  Aj"6 
   ( (   Aj"  (h"  (d"kAu"I
    ¥8"    (hG
    ”AÝC«" (Aj"6 E
    “E
 ´# AÀ k"$ @@@  (("
 A ! A60 Aìœ6,  ),7A !  Ajû"E
  ("E
  Aj"6@ 
   ( (   A6< Aú‰68  )87  Ajö"AH
  ("E
 A ! A H
   I
    kK
 @  (t"E
   ( ( 
 B 70  A,jAj"6, A 6( B 7  A j ö@@@@ A€€€€O
  At"ÝC" j!	 ( "
 j! !@ 
( ! !
 !@ (0"E
 @@  "("N
  !
 ( "
@  H
 A ! ("
  Aj!
AÝC" 6 B 7   6 
 6 @ (,( "E
   6, 
( ! (0 È  (4Aj64   Ù"E
@@ (Aj   ( (    6@@ ($" ((O
  A 6  (!
 A 6 ( !  
6 @ E
  ("
E
	  
Aj"
6 

   ( (   Aj! A j Aj÷!  6$@@  	O
   6  Aj!  kAu"
Aj"A€€€€O
@@ 	 k"Au"   KAÿÿÿÿ AüÿÿÿI"
 A ! A€€€€O
 AtÝC!  
Atj"
 6  At! 
!@  F
 @ A|j" A|j"( 6   G
   j!	 
Aj!@ E
   âC ! (! A 6@ E
  ("E
  Aj"6 
   ( (   
Aj"
 G
    AjA A>  kAugAtk  F"Aø@@ E
  
 !@  (  ( (  Aj" G
   
  !@   ( ó Aj" G
 @ ($ ( "G
 A!A !@@  Atj( "E
   (Aj"6 E
    j ë"E
 Aj" ($ ( "kAuI
  Å6 ð A !@ E
   	 kâC@ ( " E
   !@   ($"F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (     G
  ( !   6$  (( kâC A,j (0ù AÀ j$   ¯@@@   ("  ( "kAuM
  A€€€€O
  (! At"ÝC" j!   kj!@@  G
  ! !@ A|j"( ! A 6  A|j" 6   G
   (!  (!  ( !   6   6   6 @  F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
  E
    kâCÅ6 ‡@@@  (  ( "kAu"Aj"A€€€€O
 A !@  ( k"Au"   KAÿÿÿÿ AüÿÿÿI"E
  A€€€€O
 AtÝC! ( ! A 6   Atj" 6   Atj! Aj!@  ("  ( "F
 @ A|j"( ! A 6  A|j" 6   G
   (!  ( !   6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
 @ E
    kâC Å6 ð ë@@ A|j! Atj! Axj!@@@@@@   "kAu"	  A|j"	( "  ( "
L
   6  	 
6  Aj"	 ( "  	( "
   
H6     
   
J6  A|j"  Aj"
( "  ( "  H6  
 ( "
    J" 
 H6   
  
 J6    	( "  ( "  H6  	 
( "     J"   H6  
      J6  Aj" ( "	 ( "  	  H6   	   	  J6  A|j"	 Aj" ( "
 	( " 
 H6    
  
 J6  	 	( " Aj"
( "
  
H6     
  
J"
  ( " 
 H6  
 
(   
 J6  	 ( " 	( "
  
H6     ( " ( "	  	H"  
  
J"  H6   	 
( "   	  	J"
  J6  
    J" 
   
  H"	  	H6    	  	J6 @ 	AJ
   F Aj"	 Fr! @ AqE
   
 ! @ 	"
!
@  Aj( "  ( "L
 @@ 
 6 @  "	 G
  !	 	!
  	A|j" ( "J
  	 6  
!  
Aj"	 G
    
@ 	"!	@ Aj( " ( " L
 @ 	  6  "
!	  
A|j"( " J
  
 6  ! Aj"	 G
  @ 
   F
    €  	AvAtj!  ( !
@@ 	AI
 @@  ( "	 ( "J
  
 	L
   
6   	6   ( "	 ( "
L
  	6    
6 @@ 
 	L
   
6   	6    6  ( "	 L
   	6   6  ( !	@@  A|j"
( " Aj"( "
J
  	 L
 
 	6   6  
( "	 ( "L
  	6  
 6 @@ 	 L
   	6   6  
 
6  ( "	 
L
 
 	6   
6  ( !	@@  Aj"( " Aj"
( "J
  	 L
  	6   6  ( "	 
( "L
 
 	6   6 @@ 	 L
  
 	6  
 6   6  ( "	 L
  	6   6  ( !@@@  ( "
 
( "	J
 @  
J
  
!	   6   
6   	L
 
 6    	6 @  
L
  
 6   	6  
!	 
 
6    	6   	L
   6   	6  !	 ( !
  	6    
6 @ ( "	  ( "J
  
 	L
  
6   	6  ( "	  ( "
L
   	6   
6 @@ 
 	L
    
6    	6   6  ( "	 L
  	6   6  Aj! ( !	 !@ Aq"
  ! A|j(  	J
  ! @@ 	 ( L
 @ 	  Aj" ( L
  @  Aj"  O
 	  ( L
  !
@   O
 @ 	 
A|j"
( J
 @   
O
  
( !  ( !@   6  
 6 @ 	  Aj" ( "L
 @ 	 
A|j"
( "J
    
I
 @   A|j"
F
   
( 6  
 	6 A !@ "Aj"( " 	J
  !@@  G
 @  O
 A|j"(  	L
  @ A|j"(  	L
 @  O
  ( !
 !  !
@   
6  
 6 @  "Aj" ( " 	J
 @ 
A|j"
( "
 	L
    
I
 @  F
   ( 6   	6 @  I
    !	@ Aj"   E
  ! !  	E
 	
     ø Aj! A !   A|j"	 Aj" ( "
 	( " 
 H6    
  
 J6  	 	( "
 ( " 
 H6    
  
 J"
  ( "	 
 	H6   (  	 
 	J6 % @ E
    ( ù   (ù AâC   A 6  Aô¤6      A 6  Aô¤6        A 6  A”¥6      A 6  A”¥6   	   AâC‰@   F
 @   k"Au"AH
  A~jAv"!@@  "At"AuH
    j!	   Au"
Ar"Atj!@@ 
Aj" H
  ( !
 Aj"
  ( "
 
( "
J"! 
 
 
 
H!
   ! 
 	( "J
 @@ ! 	 
6   H
   At"
Ar"Atj!@@ 
Aj"	 H
  ( !
 Aj"
  ( "
 
( "
J"! 
 
 
 
H!
 	  ! !	 
 L
   6  Aj! A J
  !@  F
   Aj!  Aj! A~jAv! AH! !@@ ( "
  ( "L
   6    
6  
  ( !
A!@@ AG
  !AA 
 ( "	J"!   ! 
 	 
 	H!
  !	 
 
J
 @@ ! 	 
6   H
   At"
Ar"Atj!@@ 
Aj"	 H
  ( !
 Aj"  ( "
 ( "J"! 
  
 H!
 	  ! !	 
 
L
   
6  Aj" G
 @ AH
 @ "A~jAv!  ( !A !  !
@ At"Ar!	 
 Atj"Aj!@@ Aj" H
  ( ! 	!  	 ( " Aj"( "
J"!   !  
  
H! 
 6  !
  L
 @@  A|j"G
   6   ( 6   6    kAjAu"AH
    A~jAv"Atj"
( " ( "L
 @@ 
!	  6  E
 	!   AjAv"Atj"
( " J
  	 6  Aj! AJ
  ! …A!@@@@@@   kAu A! A|j"( "  ( "L
   6   6 AAq A|j"  Aj"( " ( "  H6      J6   ( "  ( "  H6      J" ( "  H6     (    J6 AAq  Aj"  ( " ( "  H6       J6  A|j"  Aj"( " ( "  H6    ( "    J"  H6       J6   ( " ( "    H6   ( "      J"  H6      J6 AAq  Aj"  ( " ( "  H6       J6  A|j"  Aj"( " ( "  H6      J6   ( "  Aj"( "  H6      J" ( "  H6   (    J6   ( " ( "  H6   ( "  ( "  H"	    J" 	 H6     ( "    J" J6   	  	 J"     H"   H6       J6 AAq  Aj"  Aj"( " ( "  H"  ( "  H6        J"  J"6       J 6 @@  Aj" F
 A !	@ !@ ( " ( "L
 @@  6 @ "  G
   ! !  A|j"( "J
   6  	Aj"	AG
  Aj F!A ! ! Aj"!  G
 A!  r! Aqƒ@@   Aj"F
  ( " ("O
 ( ! !@@   ( F
 @@ 
  ! @    ("( F! !  
   ! @  "(" 
  ( ( "O
@ 
   6    6  Aj@ ( " 
   6   !@@@   "(" O
  ! ( " 
   O
 Aj! (" 
   6  @  O
 @@ ("
  ! @    ("( G! !  
   ! @  "( " 
 @@  F
   (O
@ 
   6  Aj  6  @ ( " 
   6   !@@@   "(" O
  ! ( " 
   O
 Aj! (" 
   6    6   6      6   J  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     # Ak"$ @@ E
 @@@@ õ
  ëE
   ( ( 6    AjÏ6  (!  A 6  E
  ^  (Aj"6 E
@ å"E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (     6  (" E
    Aj" 6  
  ( (     A 6  Aj$ ŠA!@@  ( " E
   A ¢" E
 @@  ïE
     ( ( !@  é
 A!   (ß!  ("E
   Aj"6 
     ( (    Ž	}A !  A 6  B 7 @@@ ( "E
  ( (kA	I
 A!@  Ÿ!@@   ("O
   8  Aj!   ( "kAu"Aj"A€€€€O
@@  k"	Au"
  
 KAÿÿÿÿ 	AüÿÿÿI"
 A !	 A€€€€O
 AtÝC!	 	 Atj" 8  	 Atj!	 !@@  G
  !@ A|j" A|j"* 8   G
   (!  ( ! Aj!   	6   6  E
    kâC   6 Aj" ( "( (kAuI
 Å6 ð Î# Ak"$ A !@@  ( " E
 A!  A¢" E
      ( ( 6@ AjAó‰
 @ AjA˜Œ‰E
 A!@ AjA³€‰E
 A!@ AjA’ô‰E
 A!@ AjA€û‰E
 A!@ AjAÒ…‰E
 A!@ AjAÉ€‰E
 A!AA  AjAÄô‰! (! A 6@ E
  ^  ("E
   Aj"6@ 
     ( (   ! Aj$   §}# Ak"$ A ! A :   A :   A :  @@  ( "	E
  	( 	(kAI
  	A¢"
E
 @ 
ë"	E
  	 	(Aj"6 E
 
("E
 
 Aj"6@ 
  
 
( (   	E
   	Ü6 AjAó‰! (!
 A 6@ 
E
  
^@ E
 A !
A !@  ( A¢"E
 @ ï"E
   (Aj"
6 
E
 ("
E
  
Aj"
6 

   ( (  @  ( A¢"E
 @ ï"
E
  
 
(Aj"
6 
E
 ("
E
  
Aj"
6 

   ( (  A !@  ( A¢" E
 @  ï"E
   (Aj"
6 
E
  ("
E
   
Aj"
6 

     ( (    A G:    
A G:    A G:  @ E
   ø8 @ 
E
   
ø8 @ E
 @@ ø"C    \
  A :    8  (" E
   Aj" 6  
   ( (  @ 
E
  
(" E
 
  Aj" 6  
  
 
( (   E
  (" E
   Aj" 6  
   ( (   	(" E
 	  Aj" 6  
  	 	( (   Aj$   \A !@  ( "E
  ( (kAI
   ˆ!  ( " (  (kAuA~j"  Aœ¥j-  "   I!  @  ( " 
 C       AjŸ    6   J  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     ‰# Ak"$ A ! A 6Œ@@@@@@@  ( é"E
  A6ˆ A„„6„  )„7H@  AÈ jï" E
 @  õ"E
   (Aj"6 E
  ("E
   Aj"6@ 
     ( (   E
   Ë6€ AŒj A€j‘ (€!  A 6€@  E
   e !@@ (Œ" E
   (
 A6| Aû$6x  )x7@  AÀ jï"E
 @ õ" E
     (Aj"6 E
 ("E
  Aj"6@ 
   ( (    E
 @@  È"
  B 7p (! B 7p E
   6t  Aj6p  )p78  A8j¥6€ AŒj A€j‘ (€! A 6€@ E
  e@ E
  ^  ("E
   Aj"6 
     ( (   A6l AÃú6h  )h70   A0jñ6€ A€jA°ÿ‰! (€!  A 6€@  E
   ^ (Œ! @ E
  A 6Œ@  E
   (
 A6d Aµú6`  )`7(@  A(jï"E
 @ õ"E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   
 A6d Aèï6`  )`7 @  A jï"E
 @ õ"E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   
 A6d Aøƒ6`  )`7  Ajï"E
@ õ"E
   (Aj"6 E
 ("E
  Aj"6@ 
   ( (   
  ( õ" E
@@  È" 
  B 7P  (! B 7P E
   6T   Aj6P  )P7  Aj¥6€ AŒj A€j‘ (€! A 6€@ E
  e  E
  ^@@ È"
  B 7X (! B 7X E
   6\  Aj6X  )X7  Aj¥6€ AŒj A€j‘ (€! A 6€@ E
  e@ E
  ^ ("E
  Aj"6 
   ( (  @ E
  ("E
  Aj"6 
   ( (   
@ (Œ" 
 A ! @  (
 A !     ( Aj6  (Œ! A 6Œ E
  e Aj$    ˆ# AÐ k"$ A !@@  ( é"E
  A6L A¯„6H  )H7   A jû"E
  AÃú6< A6@  )<7   Ajñ6D AÄ jA°ÿ‰! (D!A !  A 6D@ E
  ^AA !@@@@  AtA¨¥j( "
 A ! ì7!  68  64  )47@  Ajó"E
  (! e E
   60  6,  ),7  Aj‚"
  Aj"  G
 A ! (" E
   Aj" 6  
   ( (   AÐ j$   ±# Ak"$ @@@  " 
 A !  à! A6 AÍ–6  )7   û! ("E
  Aj"6@ 
   ( (    ("E
   Aj"6 
     ( (   Aj$   Þ# Ak"$ @@@  "
 A !  à! A6 AÍ–6  )7   û!  ("E
  Aj"6@ 
   ( (   ("E
  Aj"6@ 
   ( (  @  
 A ! @  (Aj     ( (   Aj$    0A !@  ( " E
   (E
     ( Aj6   !     6   J  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     ß# A0k"$   ( ! A6, A¯6(  )(7A !@  AjÕE
   ( ! A6  Aþú6  )7   Ajô" 6$@  
 A !A !@  (E
 @ A$jAÄ«‰E
 A!@ A$jA…û‰E
 A!@ A$jAº„‰E
 A!@ A$jA¦Ä‰E
 A!@ A$jAÖæ‰E
 A!@ A$jA¦€‰E
 A!@ A$jA€Þ‰E
 A!@ A$jA³Õ‰E
 A!@ A$jAÇÛ‰E
 A	!@ A$jA…ä‰E
 A
!@ A$jAè³‰E
 A!@ A$jAó³‰E
 A!@ A$jA¸ñ‰E
 A
!@ A$jAµ‰‰E
 A!@ A$jAíË‰E
 A!@ A$jA‘®‰E
 A!@ A$jA›–‰E
 A!AA  A$jAª‡‰! ($!  A 6$  E
   ^ A0j$  `# Ak"$ @@ •A|jA|K
   A ƒ ( ! A6 Aí„6  )7      ï… Aj$ ó# AÐ k"$ A !@@  •"A
K
 A tAœÐ qE
   ( ! A6L Aû$6H  )H7 @  A jï"E
  AÄ j Œ" Ž!   AG
   ( !  A6@ AÁ°6<  )<7   Ajû" E
  A60 Aû$6,  ),7@@   Ajñ"E
  (! B 74@ E
   68  Aj64  )47 Aj¥! ^ B 7  B 74 ¥!  ("E
   Aj"6 
     ( (   AÐ j$   Ç# Að k"$ @@@  •AF
  A 6\  ( !  A6X A¦€6T  )T7(    A(jñ6\ ((!  A6P A¦€6L  )L7    A jû" E
  A€ˆ6< A6@  )<7 AÄ j AÜ j AjA p@@ - HAG
  (D
 A68 AçÌ64  )47   Ajï"E
 @@ õ
  óE
@@  ( ( "
  B 7h (! B 7h E
   6l  Aj6h@@ (\"
  B 7` (! B 7` E
   6d  Aj6`  )h7  )`7  AÜ j A0j Aj þ"„ ( ! A 6 @ E
  ^ E
  ^ ("E
  Aj"6 
   ( (    ("E
   Aj"6 
     ( (   (\!  Að j$    B# Ak"$   ( !  A6 A´ƒ6  )7    Aõ!  Aj$   @# Ak"$   ( !  A6 Aâþ6  )7    ñ!  Aj$   @# Ak"$   ( !  A6 AŸ™6  )7    ö!  Aj$   @# Ak"$   ( !  A6 AÆ6  )7    ˆ!  Aj$   ª# AÀ k"$   A 6  B 7 @@ ( "E
  A68 Aþú64  )47   Ajñ6< A<jAÇÛ‰! ( !@@ E
  A6, A—ú6(  )(7   ï! A6$ AÆ6   ) 7  Ajÿ!  60@ E
 @@@@@ é
  õE
  ("  (O
 A 60  6    Aj6A !@ å"E
  ( (F
 A !@   ¢"6@ E
 @@  ("  (O
  A 6  (! A 6 ( !  6 @ E
  ("E
  Aj"6 
   ( (   Aj!   Ajž!   6 (! A 6 E
  ("E
  Aj"6 
   ( (   Aj" ( (kAuI
   A 60   A0jž! (0!   6 A 60 E
 (" E
   Aj" 6  
   ( (   (<!  A 6<  E
   ^ AÀ j$  ‡@@@  (  ( "kAu"Aj"A€€€€O
 A !@  ( k"Au"   KAÿÿÿÿ AüÿÿÿI"E
  A€€€€O
 AtÝC! ( ! A 6   Atj" 6   Atj! Aj!@  ("  ( "F
 @ A|j"( ! A 6  A|j" 6   G
   (!  ( !   6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (    G
 @ E
    kâC Å6 ð †# Ak"$ @@@@ ( "E
  A6 A¹ú6  )7   ï"E
 @@@ õ
  óE
 ! (Aj"
A ! ("E
  Aj"6@ 
   ( (   
  A :   A :    ( ( !  A:    6  (" E
   Aj" 6  
   ( (   Aj$  ö# Ak"$ @@@  ( " E
  A6 A¹ú6  )7    ï" E
 @@@  õ
   óE
  !  (Aj"
A !  ("E
   Aj"6@ 
     ( (  @ 
 A !   ( ( !  ("E
  Aj"6 
  ( (  A !  Aj$    á# A k"$ A !@@  ( "E
  A6 A¬ˆ6  )7  AjˆE
   ( !  A6 A¬ˆ6  )7    ï" E
 @@  éE
 A!@  å"
 A ! ( (kAu!  ("E
   Aj"6 
     ( (   A j$   # A k"$ @@@ ( "E
  A6 A¬ˆ6  )7  Ajˆ
  A 6  ( ! A6 A¬ˆ6  )7 @@@  ï"E
 @ å"E
   ¬!A ! é! 
 E
  (Aj"6 ! 
  A 6    6  (" E
    Aj" 6  
  ( (    A j$     6   J  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     r# Ak"$ @@  ( "
 A ! @@ AtA¼¥j( " 
 A !  ì7!  6   6  )7   ˆ!  Aj$   x# Ak"$ @@ ( "
 A !@@ AtA¼¥j( "
 A ! ì7!  6  6  )7   û!   “ Aj$    A
IAŒ   Aÿ?qvq   AI":    A  :  6~# Ak"$   ) "7   7   A ª! Aj$  Á~# A0k"$ A !@@  E
  A J
   ) "7  7(   Ajï"
  A6$ Aï$6   ) 7   Ajû!   7   7    Ajª!  E
   ("E
   Aj"6 
     ( (   A0j$   ª# Að k"$ A ! A 6T  AÈ jAj"6H B 7L@@  E
 A !@ ! !@@ E
 @@   "("O
  ! ( "
   O
 ("
  Aj!AÝC" 6 B 7    6  6 @ (H( "E
   6H ( ! (L È  (PAj6P A6@ Aí$6<  )<7(    A(jó"6D@ E
  ("E
 @@ (T"E
  (
 AÔ j AÄ j‘ A.6X B 7d@ Aÿÿÿÿq"E
   6h  Aj6d A6`  )d7   AØ j6\  )\7@@@ Aì j A j Aj( "
 A !A ! (! B 7d Aÿÿÿÿq"E
 Aj!  6h  6d@@@ (T"
 A !A ! (! B 7\ Aÿÿÿÿq"E
 Aj!  6`  6\  )d7  )\7  Aì j Aj Aj( 6X AÔ j AØ j‘ (X! A 6X@ E
  e E
  e A68 Aï$64  )47 @   û" E
   ("E
   Aj"6 
     ( (   !@@ (L"E
 @ " " (  I"!  Atj( "
   F
      (O
 ! (D! A 6D@ E
  e (L!  E
  F
  AÈ j Ô (T! Að j$   (   B 7    6   6  AjA :    ­  Â# A k"$   (! A6 AÃ¯$6  )7A !A !@  AjA ª"E
   ( ( !  6  (! A6 Aö$6  )7 @@  A ª"E
   ( ( ! ("E
  Aj"6 
   ( (     AvAq:    AvAq: @@ AjA¦‰E
 @ A€€qE
   A6    AvAq: @ A€€qE
   A6   A:   A6 @ AjA”¦‰E
 @ A€€À qE
   A6 @ A€€€qE
   A6   A6 @ AjA—¦‰E
 A!@ A€€q
    AvAq: A!   6     ®:  AjAÆ¯$‰E
   A	6  (!  A 6@  E
   ^@ E
  (" E
   Aj" 6  
   ( (   A j$   
# AÀ k"$ @  ( AyjAO
   (!A! A6( A±€6$  )$7A !@  AjA ª"E
   (! A6( Aù$6$  )$7@  AjA ª"E
 @@@ å"E
  ( (kAu!A! ï
 A ! B 7(  A(j"6$@@@ å"E
 @ ( (kAu F
 A ! A j À"	( "( E
@ (" ("
F
 @@ ( õE
   ( " ( ( 6  Aj64 A8j A$j AjAý$ A4j A3j± (8" (Aj6 (! A 6 E
  e Aj" 
G
  	Â õE
 @ AF
 A !   ( ( 68 A$j A8j²" ( Aj6  (8! A 68 E
  e  ³!@ E
  A8j À"( "( E
@@ ("
 ("F
 A !@ 
( ïE
 
( " ( ( " O
A !    A ´"
64 !@ (("E
 @ "  Aj"	 A4j–"
! AA  
j( "
 A !@  F
  A4j Aj 	 
–
   (Aj"6A! 
  !@@ ("
E
 @ 
"( "

  @  ("( G!
 ! 

 @ ($ G
   6$  (,Aj6, (( ï (! A 6@ E
  e AâC (4!
 A 64@ 
E
  
e@ E
  
Aj"
 F
A ! Â (,E! Â@  ( ( " I
 A !    A ´68@@ (("E
  !@ "  Aj"	 A8j–"
! AA  
j( "
   F
  A8j Aj 	 
–E
 ! (8! A 68@ E
  e  G! A$j ((µ ("E
  Aj"6 
   ( (   ("E
  Aj"6@ 
   ( (   ! AÀ j$   Q  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    A 6   ‹# Ak"$   (!  A6 Aö$6  )7 A !@@   A ª" E
     ( ( !  ("E
   Aj"6 
     ( (   Aj$    Aj!@@@ ("
  !@@  "Aj"–E
  ! ( "
@  –E
  Aj! ("
A ! ( "
AÝC! ( ! A 6 ( ! A 6  (!  6@ E
  e  6 B 7  A 6  6  !@ ( ( "E
   6  ( ! ( ÈA!  (Aj6   :    6 A# Ak"$   6 Aj   Aý$ Aj Aj± (! Aj$  Ajû# Ak"$ @  ( "AK
 A tAŒqE
   (!  A6 AÀ‰6  )7 A !@   A ª"E
 @ å" E
     (Aj"6 E
 ("E
  Aj"6@ 
   ( (    E
   (  (kAu"AL
  ("E
   Aj"6 
     ( (   Aj$   ‰# Ak"$ @  ( "AK
 A tAŒqE
   (!  A6 AÀ‰6  )7 A !@   A ª"E
 @ å" E
     (Aj"6 E
 ("E
  Aj"6@ 
   ( (    E
 A !@   ¢"E
 @@ å"
  !  ¢! ("E
  Aj"6@ 
   ( (   E
@@ õ"
 A ! Ë! ("E
  Aj"6 
   ( (    ("E
   Aj"6 
     ( (   Aj$   ? @ E
    ( µ   (µ (!  A 6@  E
   e AâC
   («=~# Ak"$   (!   ) "7   7   A ª! Aj$  '@  (" E
     (Aj"6 
    Æ~~# AÐ k"$ @@@@@  ( A~j    (  è"(" ( "kAu"A H
@  F
  A AK!A !@     (  è(  Atj( ¿A º Aj" G
   (  ú  A » A 6H@  ¼"A H
     A´6D AÈ j AÄ j‘ (D! A 6D E
  e@@@@  ( Ayj   (   AÈ jö
  (   AÈ jøE
   A ½@@  ( Ayj   (  ù  (  ÷ (H! A 6H E
 e A 6D A 6@  (! A6L A ¦6H  )H7(@  A(jA ª"E
    ( ( 6H AÄ j AÈ j‘ (H! A 6H E
  e  (! A6L Aù$6H  )H7 @  A jA ª"E
    ( ( 6H AÀ j AÈ j‘ (H! A 6H@ E
  e ("E
  Aj"6 
   ( (  @ E
  ("E
  Aj"6 
   ( (    (! A6L Aš¦6H  )H7A !@@@  AjA ª"E
  ("E
  Aj"6 
  ( (  B !B !@ (D"E
  AjA  5Bÿÿÿÿƒ"§!@ (@"E
  (Aÿÿÿÿq"­B † AjA  ­„!  B ˆR
   § §At°7E
  (   AÄ jöE
   (! A6L A ¦6H  )H7@@@@  AjA ª"E
   ( ( "E
  ( AÈ jAù$ü" ‹ ( ! A 6 @ E
  ^@ E
   ( AÈ jAš¦ü"  ( ( ‹ ( ! A 6  E
  ^ ("E
  Aj"6 E
  (! A6< Aù$68  )87@  Aj"E
  ("E
  Aj"6 
   ( (    (! A64 Aš¦60  )07   "E
 ("E
  Aj"6 
  ( (    (  ÷ ("E
  Aj"6 
   ( (   (@! A 6@@ E
  e (D! A 6D E
  e AÐ j$  Ë~~# A0k"$ @@  (  è(  Atj( "E
 @ 
  ¾E
 ½!  (  è"(" ( "	kAu"A H
@  	F
  A AK!
 Aj!A !@  (  è(  Atj( !@@  - AG
 B !@@ ½"	
 A !
B ! 	AjA  	5Bÿÿÿÿƒ"§!
@ E
  (Aÿÿÿÿq"­B † A  ­„!@@@  B ˆR
  
 § §At°7
 B ! »!
 »!@@ 

 A !B !A  
Aj 
5"P!@ E
  ("­B † AjA  ­„!A !@  B ˆR
   § §°7E!@ E
  ^@ 
E
  
^  rE
  q!
A !
 E
  
À 	E
 	e@  G
   À E
  A À Aj" 
G
   (! A6, AÀ‰6(  )(7@@  AjA ª"E
   (Aj"6 E
@ å"E
   (Aj"	6 	E
 ("	E
  	Aj"	6@ 	
   ( (   E
  ("	E
  	Aj"	6@ 	
   ( (   E
  (!	 A(jAù$ü!  ù6$@ 	  A$j¿"	E
  	("E
 	 Aj"6 
  	 	( (   ($!	 A 6$@ 	E
  	^ ( !	 A 6  	E
 	^@@ 
  B 7 (! B 7 Aÿÿÿÿq"E
   6   Aj6  )7  AjÀ"6$@@ E
 @  ( A(jAù$ü" A$jô"	E
  	("E
 	 Aj"6 
  	 	( (   ( !	 A 6 @ 	E
  	^ ($!A ! A 6  (!	 A6, Aù$6(  )(7 @ 	 A ª"	E
  	("
E
 	 
Aj"
6@ 

  	 	( (    	 	( ( 6( Aj A(j„ ((!	 A 6( 	E
  	^B !B !@ ("	E
 A  	Aj 	5"P!@ E
  ("
­B † AjA  
­„!@  B ˆR
   § §°7
   (!	@ 	 A(jAù$ü" 	AjAÔÇÀþ"	E
  	 	("
Aj"
6 
E
 	 
6 

  	 	( (   ( !	 A 6 @ 	E
  	^ (!	 A 6 	E
  	^ A 6$ E
  ^@ E
   (  ú@ E
  (" E
   Aj" 6  
   ( (   E
  e A0j$  ¾# A0k"$ @@@ E
  A 6,@  A Â"A H
     A´6( A,j A(j‘ ((! A 6( E
  eA!@@@  ( Ayj   (   A,jø!  (   A,jö! (,!A ! A 6,@ E
  e E
  (! A6$ Aù$6   ) 7@  Aj"E
  ("E
  Aj"6 
   ( (    (! A6 A±€6  )7@  Aj"E
  ("E
  Aj"6 
   ( (  A! E
 @@  ( Ayj   (  ù  (  ÷ A0j$   ü
~~# Ak"$ @  ( AyjAO
   (! A6 A ¦6  )7 @@  A ª"
 A!@@  ( ( "
 A!A!@ (E
   ³E
  Aj!A !@   A ´! 5Bÿÿÿÿƒ!@@ 
 B !	 (Aÿÿÿÿq"
­B † AjA  
­„!	A !
@ 	B ˆ R
  A  §"
 	§ 
At°7E!
@ E
  e@ 
E
  ! Aj"  ³I
  e ("E
  Aj"6 
   ( (   Aj$   ð# Ak"$ @  ( AyjAO
 @ A H
    ³O
     A ´6@@ E
 @@  ( Ayj   (   Ajö
  (   AjøE
    AjÄ@  - 
     ®:  E
 @@  ( Ayj   (  ù  (  ÷ (!  A 6  E
   e Aj$  ' @  (  è" (  ( kAu" AJ
    §# Ak"$ AÝC!   ("6@ E
   ( Aj6   Aj ×" (Aj"6@ E
  (! A 6@ E
  ë@    þ"E
   (Aj" 6  E
 Aj$   ¤# Ak"$ AÝC!   ( " 6@  E
     ( Aj6   Aj Aj ü"×"   (Aj"6@ E
  ( ! A 6 @ E
  ^ (! A 6@ E
  ë Aj$       (  è(  Atj( —~~# A k"$ @  ( AyjAO
   (! A6 Aù$6  )7@@@@  AjA ª"E
  (Aj  ( AyjAO
  (! A6 A±€6  )7   A ª"
A!  ( (   ("E
  Aj"6@ 
   ( (  @ ïE
   ( ( ! A 6@@@ õE
 A! 
   ( ( 6 Aj Aj‘ (! A 6 E
 eA! å! A H
 E
A !@  ¢"E
   ( ( !  6 Aj Aj‘ (! A 6@ E
  e E
  ("E
  Aj"6 
   ( (  @   ÓO
 A !B !B !@     Ô"A ´"E
  AjA  5Bÿÿÿÿƒ"§!@ ("E
  (Aÿÿÿÿq"­B † AjA  ­„!@@  B ˆQ
 A!A !A   § §At°7"! E!@ E
  e 
A!  ³E
 A !@B !   A ´!B !A !@ ("E
  AjA  5Bÿÿÿÿƒ"§!@ E
  (Aÿÿÿÿq"­B † AjA  ­„!A !@  B ˆR
   § §At°7E!@ E
  e@ E
  ! Aj"  ³I
  (! A 6 E
  e A j$       A´ú# Ak"$ @@@  ( AF
   (! AjAù$ü!@@ ( "
  B 7 (! B 7 Aÿÿÿÿq"E
   6  Aj6@   Ajò"E
  ("E
  Aj"6 
   ( (   ( ! A 6 @ E
  ^  (! AjA±€ü! A$ÝC Aj–" (Aj"6 E
@    þ"E
   (Aj"6 E
  ( !  A 6 @ E
  ^AÝC ñ"   (Aj"6 E
@   ¼" E
     ("Aj"6 E
   6 
     ( (   (" E
   Aj" 6  
  ( (     Ù  (!@  - 
  AjAù$ü! @@ ( "
  B 7 (! B 7 Aÿÿÿÿq"E
   6  Aj6@    Ajò"E
  ("E
  Aj"6 
   ( (    ( !  A 6  E
 ^ AjAù$ü!A$ÝC Aj–" (Aj"6 E
 @   þ"E
   (Aj"6 E
 ( ! A 6 @ E
  ^@  ³E
 A !@@@  F
    ÖE
A !A !@@   A ´"E
  (! B 7 Aÿÿÿÿq"E
 Aj!  6  6@  AjÚ"E
  ("E
  Aj"6 
   ( (   E
  e Aj"  ³I
  E
 ("E
   Aj"6 
  ( (    Aj$ c@ 
 A  (  è"( "! @  ("F
  ! @  (  F
  Aj"  G
  ! A   kAu   F,A !@  ( Aj" AK
   A£¦j-  ! Aÿq‰# Ak"$  (! A6 AÜ¯$6  )7 @@@  A ª"
   A £   à£ (" E
   Aj" 6  
   ( (   Aj$  ‹# Ak"$   (!  A6 A¦6  )7 A !@@   A ª" E
     ( ( !  ("E
   Aj"6 
     ( (   Aj$   ˜# Ak"$   (! AjAö$ü! AÝC ñ" (Aj"6@ E
 @    þ"E
 @ (Aj   ( (    ( !  A 6 @ E
  ^ Aj$  š# A k"$ @@@  ( A~qAG
    Ë!@@ E
   (! A6 A ¦6  )7  AjA ª!  (! A6 Aù$6  )7  AjA ª!@ 
 A ! 
  ( AF
  (! A6 A ¦6  )7 A !  A ª"E
A !@@@  ( ( A}j    ( ( !A ! åA ¢" E
     ( ( !  ("E
   Aj"6 
     ( (   (" E
   Aj" 6  
   ( (   A j$   Š# A k"$  A6 AÔÇ6  )7  Aj£"6@  (  è"(" ( "kAu"A H
 @  F
  A AK!A !@  (  è(  Atj( !@@@ E
  ¿
 ¾E
  ½6 Aj Aj‘ (! A 6@ E
  e (! Aj" G
  A j$   	   A Êà# A0k"$ @@@  ( "AK
 @A t"Aðq
 @ Aq
  AG
A !   Î"A H
@ E
    ¼F
@ E
   (   øE
@ 
   A »   A ½ E
  (  ù     Ï  ( "6,@ E
   ( Aj6 @@ E
 A!  (   A,jöE
 A(jA ¦Aù$ ü!  (!@@ (,"
  B 7  (! B 7  Aÿÿÿÿq"E
   6$  Aj6 @   A jò"E
  ("E
  Aj"6 
   ( (  @@@  ( "AG
    A,jÎ"AJ
  ( !@ 
  AG
   (! A jAš¦ü!  (!@@ ( "
  B 7 (! B 7 E
   6  Aj6  )7    Ají" ( ( ‹ ("E
  Aj"6@ 
   ( (   ( ! A 6  E
  ^  (! A6 A±€6  )7   "E
 ("E
  Aj"6 
  ( (   
   A »   A ½@ E
   (  ÷ ( ! A ! A 6   E
   ^ (,! A ! A 6,@  E
   e 
A! A0j$   Ï~~@  ³E
 A !@B !B !A !@   A ´"E
  AjA  5Bÿÿÿÿƒ"§!@ ( "E
  (Aÿÿÿÿq"­B † AjA  ­„!A !@  B ˆR
   § §At°7E!@ E
  e@ E
   Aj"  ³I
 Aƒ
~~@  (  è"(" ( "kAu"A H
 @  F
  A AK!A !@B !	B !
A !@  (  è(  Atj( "½"E
  AjA  5Bÿÿÿÿƒ"
§!@ ( "E
  (Aÿÿÿÿq"­B † AjA  ­„!	A !@ 
 	B ˆR
   	§ 
§At°7E!@ 
   (  è"( "
!@ 
 ("F
 @ (  F
 Aj" G
  !  A  
kAu  F A º@ E
  e  Aj" FrAG
 @ E
   (  úA 
    A  Íñ# A0k"$   (! A6, AÉ±6(  )(7@@@  AjA ª"E
   ( ( ! (" E
   Aj" 6  
  ( (  @  (  è"( "  ("F
 @@  ( "E
 @ ("E
   (Aj"6 E
 A6$ AÉ±6   ) 7@  Ajˆ"E
  A6 AÉ±6  )7   ö! ("E
  Aj"6@ 
   ( (   
  Aj"  G
 A ! A0j$   Ø# A k"$ @@@  ( AyjAO
   (! A6 Aù$6  )7@@@  AjA ª"E
  (Aj  ( AyjAO
  (! A6 A±€6  )7A !   AjA ª"E
  ( (   (" E
    Aj" 6@  
   ( (  @@ õ
  ïE
  ( ( "
A ! @ å"
 A !  ( (kAu" AJ
  (!  ^  A G!  A j$   ð# Ak"$ @  ( AyjAO
   (!  A6 A±€6  )7 A !@   A ª"E
 @ å" E
     (Aj"6 E
 ("E
  Aj"6@ 
   ( (    E
   (  (kAu"AL
  ("E
   Aj"6 
     ( (   Aj$   š# Ak"$ A !@@ A H
   ( AyjAO
  (! A6 A±€6  )7 @  A ª"
 A!@ å" E
     (Aj"6 E
 ("E
A!  Aj"6@ 
   ( (    E
 A!  (  (kAu"AL
@  O
    ª!  ("E
   Aj"6 
     ( (   Aj$       A ´# Ak"$ @  ( AyjAO
 A !@ A H
    ³O
 @  - AG
    ×!    A ´"6   AjØ! E
  e Aj$   ³# Ak"$ @  ( AyjAO
   (! A6 A±€6  )7 A ! @  A ª"E
 @@@ å" E
  Aj  À"( "( E
@ ("  ("F
 @@  ( ïE
   ( " ( (  F
  Aj"  G
  Â@ ï
 A !   ( (  F!  ÂA!  ("E
  Aj"6 
   ( (   Aj$    À
~~# Ak"$   (! A6 Aù$6  )7 A ! @@  A ª"E
 @@ å" E
  Aj  À"( "( E
@ ("  ("F
 @@  ( õE
 B !@@  ( " ( ( "
 A !B !	 AjA  5Bÿÿÿÿƒ"	§!@ ( "
E
  
(Aÿÿÿÿq"­B † 
AjA  ­„!@ 	 B ˆR
   § 	§At°7!
@ E
  e 

 ÂA!  E
  e  Aj"  G
  Â@ õ
 A ! B !@@  ( ( "
 A !
B !	 AjA  5Bÿÿÿÿƒ"	§!
@ ( " E
   (Aÿÿÿÿq"­B †  AjA  ­„!A ! @ 	 B ˆR
  
 § 	§At°7E!  E
  e ("E
  Aj"6 
   ( (   Aj$    ¹# Ak"$   (!  A6 A±€6  )7 @@@@   " (  (F
 A !@   ª" F
@  L
 AÝC ñ" (Aj"6 E
    ¹"E
 (Aj Aj"  (  (kAuI
 AÝC ñ" (Aj"6 E
   ¼"E
 (Aj   ( (    ("E
    Aj"6@ 
     ( (   Aj$  ½~# A k"$ AÝC!   ("6@ E
   ( Aj6   ) "7  7  Aj AjÃ" (Aj"6@ E
  (! A 6@ E
  ë@   ¼"E
   (Aj" 6  E
 A j$      ( " A	IAŒ  Aÿqvq‹# Ak"$   (!  A6 A£€6  )7 A !@@   A ª" E
     ( ( !  ("E
   Aj"6 
     ( (   Aj$   ´# Ak"$   A Ajµ8A !  A j!A ! @@  AF
A A 6È¤lAã A>AÐ A (È¤l! A A 6È¤lA!@  E
 A (Ì¤l"E
 @   Aj¶8"
    ·8  ¸8¹8!  AF
 A! Aj$     A jú,µ# Ak"$   A Ajµ8  A j!A ! @@@  AG
 A !A A 6È¤lAä !A (È¤l! A A 6È¤lA!@@  E
 A (Ì¤l"E
    Aj¶8"E
 ¸8¹8!  AF
   ·8  Aj$  ±# Ak"$   A Ajµ8  A j!A ! @@@A!  AF
A A 6È¤lAå  !A (È¤l! A A 6È¤lA!@  E
 A (Ì¤l"E
    Aj¶8"E
 ¸8¹8!  AF
  Aj$     ·8 ³# Ak"$   A Ajµ8  A j!A ! @@@A!  AF
A A 6È¤lAæ   !A (È¤l! A A 6È¤lA!@  E
 A (Ì¤l"E
    Aj¶8"E
 ¸8¹8!  AF
  Aj$     ·8   A  A @@   ("(L
   (A·8   (  j6   ("   ( k6
   (A·8    ®~AèÝCÉ"A´¦6 A ! A0jA AœØ6! AÐjA AôØ6 A×jB 7   AÐjB 7  B 7ÈA! A6à  ) "	B ˆ§!
 	§! @@@@ 	BÿÿÿÿX
 @@@ !@   j"-  AÿG
    j-  AØF
 ! Aj" 
G
   
 k!
 !   
6Ì   6È 
AI
   
jA~jAÿ:   (ÌAj" 
O
   jAÙ:   Aç6¼ Aç6¬ Aè6° Aé6¬ Aè6¨ Aê6¤ Aë6  Aì6´  : Þ Aí6¸ Aî6°  6  6  6  6 AëE
 (ô" H
 (ì" I
   lAj"A|q"6@@ 
 A !A !  AL
 AV" A  Ø6 j!@ (Ð"E
   6Ô J  6Ø  6Ô   6Ð A : Ý A6  (ô6   
6Ì   6È@ - ÜAG
  Þ B 7$@ (Ð"E
   6Ô J ËAèâCA  Å6 â    A0j"6Ü    A j6Ð@ Ý
 A   A: Ü    )7ì    A¤j"6è    )È7¤@@ AàAF
 @ E
   ("Av!  (È!  (Ì!@@  (Ð(A)F  (ðAÿÿFq  (ìAÜÿIq AjAÜÿIq  (AjAÜÿIq"AG
  Aâ I
  AÙ j-  AÿG
  AÚ j-  AðqAÀG
  AÞ j-  AÿG
  Aß j-  AÿG
  Aà j-   AÿqG
  Aá j-   AÿqG
 AÞ !  A¦KqAG
 Ažj-  AÿG
 A~qAžF
 AŸj-  AðqAÀG
 A£j-  AÿG
 A¤j-  AÿG
 A¥j-   AÿqG
 A¦j-   AÿqG
A£! Þ  A : Ü  (Ì" I
  F
  (È j"  (Av:    kAM
 Aj  (:  @ Ý
 A   A: Ü   6è    )7ì    )È7¤ AàAF
 Þ  A : ÜA @  (ôE
   A: Þ@  (ôAG
   - Þ
     (ø6ü    (ì"6    (ð"6   6   6    („6àA ¼~# A k"$  ) "B ˆ§! §!@ B€€€€ T
 A !A!@@ !@  j"-  AÿG
   j-  AØF
 ! Aj" G
    k! !A ! AjA A˜Ø6 Aè6ˆ Aé6„ Aè6€ Aê6ü Aë6ø  Aøj6¨  Aj6´@@@ AjÝE
  Aç6” Aç6„ Aì6Œ Aí6 Aî6ˆ  6€  6ü  Aüj6À AjAàAF
 AjÞ  A :   (Ð! )Ä! (Ì! (ü! AjÞ   AF AFr:    6   6   7 A!   :  A j$ @@  - ÜAG
   A0jÞ  B 7$@  (Ð"E
    6Ô J  ËF@  - ÜAG
   A0jÞ  B 7$@  (Ð"E
    6Ô J  ËAèâC   (Ì  (¨ky@@@  - ÝAG
   A0jÞA !  A ëE
    (à6„    )7@  A0j"ß
  ÞA   (À  (J
A!  A: Ý  b# Ak"$   (Ð6 A0j AjAá! (Ô!  A  (Ð" AH"6   A   k 6 Aj$ :}}}@  * "  *"‘7"C·Ñ8]
     •8    •8 =  ("  ( "k s  sq  ("  (" k s   sqrAsAvJ@  ( "  ("L
    6   6 @  ("  ("L
    6   6û
 (! ( !   (" ("  J"  ("  ("  J"  H"	6      J"
  ( "  ("  J" 
 H"
6      H"    H"  J"6      H"    H"  J"6 @@  
J
   	L
  B 7   AjB 7 ’ (!    ("k  "6    k  "6  ( !    ("k  "6    k  "6@  L
    6   6 @  L
    6   60    ( ²8    (²8   (²8   (²8  ,    * 8    *8   * 8   *8  
}}}}}}@ ("
   B 7   AjB 7  ( "*! * !@@ At"AG
  ! ! Aj!	@@ AqE
  	! !	 ! ! Aj* "
   
]! 
  
 ]! Aj* "
   
]! 
  
 ]! Aj! AF
   j!@ Aj* "
 	Aj* "   ]"  
]! 
    ]" 
 ]! Aj"	* "
 * "   ]"  
]! 
    ]" 
 ]! Aj" G
    8   8   8   8 L}}@  * "  *"^E
    8   8 @  *"  *"^E
    8   8¤
}}}}}}}}}@@  * "  *"^
  ! !   8   8  !@@  *"  *"^
  ! !   8   8 ! *! * !   *" *"	  	^""
  
 ]"
8      ^""   ]"8   	  "   ]"8     "   ]"8 @@  ^
   
^E
  B 7   AjB 7 ö
}}}}}}}}}@@  * "  *"^
  ! !   8   8  !@@  *"  *"^
  ! !   8   8 ! *! * !   *" *"	  	^""
   
]8      ^""   ]8   	  "   ]8     "   ]8 Æ}AÿÿÿÿA  * Ž"C   Ï`! CÿÿÿN_!@@ C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!     "6 AÿÿÿÿA  *"C   Ï`! CÿÿÿN_!@@ C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!     "6AÿÿÿÿA  *"C   Ï`! CÿÿÿN_!@@ C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!     "6AÿÿÿÿA  *Ž"C   Ï`! CÿÿÿN_!@@ C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!     "6@  L
    6   6 @  L
    6   6Þ}}}} * "" Ž"  “ *" “  “"“‹’  “  “ “‹’^"CÿÿÿN_As C   Ï`Asr  ’"CÿÿÿN_As C   Ï`Asrr!@@ ‹C   O]E
  ¨!A€€€€x!  A   "6@@ ‹C   O]E
  ¨!A€€€€x!  A   "6  *"" Ž"  “ *" “  “"“‹’  “  “ “‹’^"CÿÿÿN_As C   Ï`Asr  ’"CÿÿÿN_As C   Ï`Asrr!@@ ‹C   O]E
  ¨!A€€€€x!  A   "6@@ ‹C   O]E
  ¨!	A€€€€x!	  A  	 "6@  L
    6   6 @  L
    6   6x}}}}}   *" *"’C   ?”"  “" *" * "“"  ^C   ?”"’8    ’C   ?”" ’8    “8    “8 p}}}A !@ * "  * "  *"  ^"_E
     `E
  *"  *"  *"  ^" _E
      `! œ}}}}A !@ *" * "  ^"  *"  * "  ^"`E
       _E
  *" *"  ^"  *"  *"  ^" `E
        _! j}}   * "  * "  ]8    *"  *"  ]8   * "  *"  ]8   *"  *"  ]8h}}}}     *"  *"  ^"’8     * "  *"  ^"’8      “8      “8 h}}}}    *"  *"  ^" “8    * "  *"  ^" “8      ’8      ’8 ï}}}}@@ *  *`
  * *`E
  B 7   AjB 7   Aj" Aj) 7    ) 7     * "  * "  ^"’"8      “"8      *"  *"  ^"’"8      “"8@  ^E
    8   8 @  ^E
    8   86      * ’8      *’8     *’8     *’86      * ”8      *”8     *”8     *”8x}}}}    *"  *"“C   ?” ”"  ’C   ?”"’8    *"  * "“C   ?” ”"  ’C   ?”"’8    “8    “8 ¿}}}} * ! *! *!@@ *"‹C   O]E
  ¨!A€€€€x!   6@@ ‹C   O]E
  ¨!A€€€€x!   6@@ ‹C   O]E
  ¨!A€€€€x!   6@ ‹C   O]E
    ¨6   A€€€€x6 H * •! *•! *•!   *•6   6   6   6 ¸}}}}}}}  B 7  B€€€€€€€À?7  B€€€ü7 @ * " *"” *" *"”“"C    [
     •8    •8     Œ"•8    •8    *"”  *"”“ •8    ”  ”“ •88A !@  * C  zD”‹  *‹]E
   *C  zD”‹  *‹]! 8A !@  *C  zD”‹  * ‹]E
   *C  zD”‹  *‹]!       *’8     *’8P      * ”8      *”8     *”8     *”8     *”8     *”8 }}}  *! Õ6!    *" á7"”  ”’8    ”  ”“8     * "”   *"”’8    ”  ”“8      *"”   *"”’C    ’8    ”  ”“C    ’8«}}}C  €?!C  €?!@ *  *“"‹Coƒ:]
  *  *“ •!   8 @ * *“"‹Coƒ:]
  * *“ •!   8   *  *  ”“8 *! *!  B 7     ”“8Q}}  * !@  *"C    \
   Œ C    ^@ C    \
   Œ C    ^  ‘7Q}}  *!@  *"C    \
   Œ C    ^@ C    \
   Œ C    ^  ‘7¨}}}}}}}}}}} *! * ! *!   *" *" *"C    ”"’’"	   ’’"
  C    ”" ’’"   ’’"  ]"  
]"  	]8     C    ”"’’"   ’’"  C    ”" ’’"   ’’"  ]"  ]"  ]8   	 
    ]" 
 ]" 	 ]8        ]"  ]"  ]8 Ê}}}}}}}}}}}}}} *! * ! *!   *" *" *"”"	 *"
 *"”"’’"
  	 *" ”"’’"	   * "”" ’’"   ’’"  ]"  	]"  
]8     ”"  
”"
’’"    ”"’’"   ”" 
’’"   ’’"  ]"
 
 ]"
 
 ]8   
 	    ]" 	 ]" 
 ]8        ]"  ]"  ]8     * ”   *”‘7Â}}}  * !@@  *"C    \
   Œ C    ^!@ C    \
   Œ C    ^!  ‘7!  *!@@  *"C    \
   Œ C    ^!@ C    \
   Œ C    ^!  ‘7!   ’”C   ?”Z}}}}} *! * ! *!   * * * "” *" *”’’8     ”  ”’’8    A 6    	    6      :     Av:    Av:    AtA€€üq  A€þqr  AvAÿqr%  AtA€€üq  Atr A€þqr AvAÿqr   B 7   AjA ;   "~ ) !   : 	   :    7       ) 7   Aj Aj/ ;         A 6  B 7   ›  A 6  B 7 @@ (" ( "F
   k"AmAÖªÕªO
   ÝC"6   6     j6@  ) 7  Aj Aj/ ;  Aj! Aj" G
    6  Å6 '@  ( "E
    6   ( kâC   @  (   (" F
   A}jA:  ²# Ak"$ @ ( " ("F
   ( !    ("    kAm§	 E
   k"  (  ( "kO
  Am!@ Aj   Al"j˜	  (  j )7  Aj"  (  ( "kAmI
  Aj$ …@ AN
  @@   ("  ("kAmJ
 @   k"AmJ
   Alj! !@@  j" G
  ! !	 !@  	) 7  Aj 	Aj/ ;  Aj! 	Aj"	 G
    6 AN
 @@   ( "kAm j"	AÖªÕªO
   kAm!
@@  kAm"At" 	  	KAÕªÕª AªÕªÕ I"
 A ! AÖªÕªO
 AlÝC!  
Alj"!	@ Al"
E
  AjAÿÿÿÿq!@@ Aq"
  !A !	 !@  ) 7  Aj Aj/ ;  Aj! Aj! 	Aj"	 G
   
j!	@ AI
 @  ) 7  Aj Aj/ ;  Aj Aj/ ;  Aj Aj) 7  A j A j/ ;  Aj Aj) 7  A$j A$j) 7  A,j A,j/ ;  A0j! A0j" 	G
   ( ! Al! !@  F
  ! !@ Atj" Atj") 7  Aj Aj/ ;   G
   (!  j!@  F
 @ 	 ) 7  	Aj Aj/ ;  	Aj!	 Aj" G
    	6  ( !   6   (!   6@ E
    kâC Å6 ð   Al"	j! !@  	k"	 O
  	! !@  ) 7  Aj Aj/ ;  Aj! Aj" I
    6@  F
 @ Atj" 	Atj"	) 7  Aj 	Aj/ ;   	G
 @  G
   !@  ) 7  Aj Aj/ ;  Aj! Aj" G
  ì	~@  ("  ("O
  ) ! A : 	  :   7    Aj6@@   ( "kAm"Aj"AÖªÕªO
 A !	@  kAm"
At"   KAÕªÕª 
AªÕªÕ I"E
  AÖªÕªO
 AlÝC!	 ) ! 	 Alj"A : 	  :   7  	 Alj!	 Aj!@  F
 @ Atj" Atj") 7  Aj Aj/ ;   G
   (!  ( !   	6   6   6 @ E
    kâC   6Å6 ð î~@  ("  ("O
  ) ! A: 	  :   7    Aj6@@   ( "kAm"Aj"AÖªÕªO
 @@  kAm"	At"
  
 KAÕªÕª 	AªÕªÕ I"
 A !	 AÖªÕªO
 AlÝC!	 ) ! 	 Alj"A: 	  :   7  	 Alj! Aj!@  F
 @ Atj" Atj") 7  Aj Aj/ ;   G
   (!  ( !   6   6   6 @ E
    kâC   6Å6 ð     *  * * *«	ä# A k"$   8  8  8  8  8  8  8  8 @@  (   ("F
  Atj*  “‹»Dü©ñÒMbP?d
  Axj*  “‹»Dü©ñÒMbP?dE
   AjA¨	   AjA ¨	@@  (   ("F
  Atj*  “‹»Dü©ñÒMbP?d
  Axj*  “‹»Dü©ñÒMbP?dE
   AjA¨	   AjA ¨	@@  (   ("F
  Atj*  “‹»Dü©ñÒMbP?d
  Axj*  “‹»Dü©ñÒMbP?dE
   AjA¨	   A ¨	@@  (   ("F
  Atj*  “‹»Dü©ñÒMbP?d
  Axj*  “‹»Dü©ñÒMbP?dE
   A¨	   AjA ¨	@  (   (" F
   A}jA:   A j$ y@ ( " (G
   B 7   AjB 7    ø!@ ( ( "kAmAI
 A! @    Alj‚	  Aj"  ( ( "kAmI
 Ë}}}}}}}}}}}}# Ak"$   B€ ¾Œ€ÔáG7  B€ ¾„€ÔáÇ 7 @@ (" ( "F
   kAm! Œ!A !	A !
@@@@@  
Al"j"- "
AG
 @ 
Aj"
 F
   kAm! - 	AG
   ‚	@ 
AG
  - 	Aq
  
Aj"
 O
   ‚	   (  jAj‚	 ( ! (!@ 
Aj"  kAm"F
   AljAj-  AG
 
Aj!
A ! 
! 
Aj!
A! 
!	 
 O
  O
@@ E
  	 O
  	Alj"* "  Alj"* "“‹!@  
Alj"* " “"‹CÍÌL=]"E
  CÍÌL=]E
  *! *!   ’8   C  €?C  €¿  ^”’"8   Aj‚	 * !  8   “8   Aj‚	@@@@@ E
 C    !C    !C    ! CÍÌL=]
 *!C    !C    !C    ! *"  *"“  “•" ”“!    “‘7” •‹! CÍÌL=]
  *" “"  “"•" ”“!   ‘7” •‹! E
      ]’"8   ” ’  Œ *  ” ’]’8   Aj‚	      ]’"8   ” ’  Œ *  ” ’]’8   Aj‚	@  “"‹CÍÌL=]E
 @  ^As  ^F
      ®	     ®	    Œ *  ” ’]’   Œ   ” ’]’"“ •"8   ” ’8   Aj‚	    
Alj  Alj ®	 
Aj"
 (" ( "kAm"I
  Aj$   }}}}}# Ak"$  *! *!@@ * " * "\
 @  \
    ’8   ’8   Aj‚	 * !  * “8   “8   Aj‚	   ’8   Œ   ]’"8   Aj‚	 * !  8   “8   Aj‚	@  \
    ’8   Œ   ]’"8   Aj‚	 *!  8   “8   Aj‚	    “"”   “"	‘7"•"   	” •"’"’8   ’" “8   Aj‚	   “8   ’8   Aj‚	 Aj$ O# Ak"$ @  ( "  (" F
 @ Aj  ˜	  )7  Aj"  G
  Aj$ e# Ak"$ @@  (  ( kAmAI
  Aj  ±	 Aj²	!  ("E
  ( kâC  ²	!  Aj$   ä	# Ak"$ @@@@@ ( "*  ("Atj* \
  * Axj* [
  A 6  B 7  AÈ ÝC"6   AÈ j"6 Aj Aj/ ;   ) 7   Aj"6@@ Aj" F
 @@  kAm"	  "kAmjAG
   6@ Aj ³	 Aj" G
  (! (! ( !@@ Aj-  
  Aj-  
  A}j-  
  *  Atj* \
  Aj*  Axj* [
@@  O
   ) 7  Aj Aj/ ;  Aj! 	Aj"AÖªÕªO
@@  k"
Am"At"   KAÕªÕª AªÕªÕ I"
 A ! AÖªÕªO
 AlÝC!  	Alj" ) 7  Aj Aj/ ;   Alj! !@  F
 @ Atj" Atj") 7  Aj Aj/ ;   G
  Aj!  6  6 @ E
   
âC !  6  kAmAK
 (! ! Aj" G
    6   6   6   A 6  B 7  E
    kâC Aj$ Å6 ð ³}}}}}}}}A !@@@@  (  ( " kAm"A|j   * "  A0j* \
  *"  A4j* [
  *!  * !  Aj* !@   Aj* "\
   [
  A(j* !  Aj* !A!	@  Aj* "
  A$j* "\
   [
@   	AljAj-  
 	Aj"	 G
 @ 
 [
   \
@  
[
   \
@  [
   \
  [  [r! ó@  ( "(" ("O
   ) 7  Aj Aj/ ;   Aj6  @@  ( "kAm"Aj"AÖªÕªO
 @@  kAm"At"	  	 KAÕªÕª AªÕªÕ I"
 A !	 AÖªÕªO
 AlÝC!	 	 Alj" ) 7  Aj Aj/ ;  	 Alj! Aj!@  F
 @ Atj" Atj") 7  Aj Aj/ ;   G
  (! ( !  6  6  6 @ E
    kâC  6  Å6 ð ¯~}}}}# AÐ k"$  ( ! (!A ! A 6L B 7D@@  kAmAO
  Aj! AÄ jAj! Aj ±	  ("6D  ("6H  ( 6L AÄ j! !@@ 
 @ ²	
   A :   A :   ) ! AjAj" Aj) 7   7 Ajú  Aj ) 7    )7   A: @@@@@  kAm"	A|j  *!
 * ! * " A0j* \
 *"
 A4j* \
@  Aj* \
  
 Aj* [
A!@ Aj*  A$j* \
  Aj*  A(j* [
@  AljAj-  
 Aj" 	F
    A :   A :   A8jB 7  A0jB 7  A(jB 7  AjAjB 7  B 7@@@  G
 C    !C    !
 Aj  ˜	  )"7@@ (  ( "kAmAO
  B ˆ§¾!
 §¾!C    !
C    !A!@ Aj   Alj˜	 Aj Atj" )"7 @ Axj"*  §¾[
  * B ˆ§¾[
   A :   A :   Aj" (  ( "kAmI
  *4!
 *!
 *0! *!  [
  
 
[
   A :   A :   AjAj" )(7   
8  8 Ajú  Aj ) 7    )7   A:  (D!@ E
   6H  (L kâC AÐ j$    B 7  AÐ¦6   AjB 7   '@  ("E
    6   ( kâC  ,@  ("E
    6   ( kâC  AâC®AÝC"B 7 AjA 6 @@  ("  ("F
   k"AmAÖªÕªO
  ÝC" 6    j6@   ) 7   Aj Aj/ ;   Aj!  Aj" G
    6 A6 AÐ¦6  Å6    	   AâC+   B 7  B€€€‰„€€À?7  A ;   AjB 7   '@  ("E
    6   ( kâC  Õ@   ("  ( "k"AuK
 @   (" k"AuM
   j!@  F
    ×6  (!  k!@  F
    ×6    j6  k!@  F
    ×6    j6@ E
    6  âCA !  A 6  B 7 @ A€€€€O
  Au"   KAÿÿÿÿ AüÿÿÿI"A€€€€O
    At"ÝC"6   6     j6  k!@  F
    Ö6    j6Å6 _@  ("E
    6   ( kâC  A 6  B 7   ( 6   (6   (6 A 6 B 7 ;   B 7  B€€€‰„€€À?7  A ;  A 6  Að¦6   AjB 7   '@  ("E
    6   (  kâC  ,@  ("E
    6   (  kâC  A$âCšA$ÝC"Aj  Aj) 7    )7 A 6  B 7@@  ("  ("F
   k" AL
   ÝC"6    j"6     Ö6  6 A6 Að¦6  Å6 \@  ( "AjA²œ ·7"
 A AÝC"A€§6    ( " 6@  E
     ( Aj6   6 =  A€§6   (!  A 6 Ð6  (!  B 7@ E
  ^  B  A€§6   (!  A 6 Ð6  (!  B 7@ E
  ^  AâCþ# Ak"$ @@  (Ú7"
 A !@@  (" 
  B 7   (! B 7  E
   6$   Aj6  A6ˆ Aƒ–6„  ) 7  )„7 Aj!@@ AŒj Aj Ajþ( "
  B 7  (!  B 7   E
    6$  Aj6   ì76ˆ  6„  ) 7  )„7  AŒj Aj þ( ! @ E
  ^@  AjA²œ   A jä7"A H
     ($A€àqA€€F:   AJ!  E
   ^ Aj$     B 7   AjA 6   AjB 7   .    ) 7   Aj Aj( 6   Aj Aj) 7      A 6   Á@@ ( "( " ("j" s  sqA H
   ( "k" s  sqAJ
  A :    A : @@ (" ("k" s  sqA H
   ("k" s  sqAJ
  A :    A :    ­B † ­„7   A: h# Ak"$ A ! A 6  Aj®    ("6 @@ 
 A ! ( ! (!   6   6 Aj$   +  ( !  A 6 @ E
 ‡((  ¯   ! @   (I
    ( AljAj( ! @   (I
    ( AljAj( ! @   (I
    ( AljAj( Û@@@  -  "Aõ G
 @  Aj-  "Aî G
   Aj-  Aé G
 A  Aj-  "AIj A¹jAzI APj" A	K"AK
 A  Aj-  "AIj A¹jAzI APj" A	K"AK
 A  Aj-  "AIj A¹jAzI APj" A	K"AK
 A  Aj-  "AIj A¹jAzI APj" A	K"AK
    At AtrrAtr!  Aj-  "E
 A.G
  A€€€€xrA AIj A¹jAzI APj" A	K"AK
 A  Aj-  "AIj A¹jAzI APj" A	K"AK
 A  Aj-  "AIj A¹jAzI APj" A	K"AK
 A  Aj-  "AIj A¹jAzI APj" A	K"AK
    At AtrrAtr!@A  Aj"-  "AIj A¹jAzI APj" A	K"AK
   Atr!A  Aj"-  "AIj A¹jAzI APj" A	K"AK
   Aj!  Atr! -  "E
  ! A.G
 A€€€€xr  !@@@@ Aÿq"A.F
  
   Ñ	   K
 Aj"-  !     Ñ	A€€€€xr! žA !@  E
    O
 Aà­%- "E
   Aj!  ,  !A ! @@Aà­%Aà­%   j"A~qj"Aj-  At Aj-  rj"-  Aÿ q" F
 Au"Aj    H""    "H
 A  -  !@  O
 @ Aj"-  "Aÿ q!  À! À! -  !@@ AJ
  ! Aÿq  F
A A !  E
 AA AJjAj! Aÿq!@ Aà­% -  At Aj-  rj"-  "Aÿ qF
 Aj!  AJ!  Aj!  
   Aj" G
 A ! ÀA H
  Aj,  AJ
  Aj-  At Aj-  r Ç~# A k"$ @@@@Aà­%- "
   (!  ) "B ˆ§!A !@ A½µ6  7  7  Aà­%6  )7 Aj A Aà­% Atj"Aj-  At Aj-  r Ó	
 Aj" G
  E
  ( A :   A j$  ®~~# A k"$  ("   K!  ("   K!	 ( !
  ( !@@ " 	F
  F
 
 j  j"
-  Aÿ q:   Aj! Aj! 
,  A H
   O
  
 jA :    O
  Aj!
@@  j,  "AJ
  
 O
 Aj"
 O
@  
j-  At  
j-  r AÿÿqG
 A! Aj!
@ Aÿ q"	
 A ! ) !  ) !A !@ At 
j" O
 Aj"
 O
  
j-  !
  j-  !  7  7  7  7  Aj   
 Atr Ó	"
 Aj" 	G
  A j$   ç# A k"$   B 7A !  AjA 6 @ ( " ("F
  AF!A !A !A !	A !
A !@@ ( E
  B 7 Aj  AjÊ	 - AG
  ( ("("Am  " ("j"
 s 
 sqA H
  (" ("j" s  sqA H
 @@ 
Aq
  ! 
! !	 !    J!  
  
J!  	  	H!	    H!A!
 Aj" G
    6   6   	6   6  A j$ Þ	# A0k"$ A !@@  ("AI
  A~q"AF
 AF
   ( " Aj-  At  Aj-  r"I
  Azj"  Aj-  At  Aj-  r"AlI
  E
    j!  k!	  Aj! A !@ AM
 A~q"AF
@@   Aj-  At  Aj-  rG
  AF
  Aj/  "At Avr!@  /  "
At 
AvrAÿÿq"
AG
  Aÿÿq
  A	M
@ Axj  A ! 	  A
j-  At  Aj-  r"  Aj/  " At  AvrAÿÿq" jI
 B 7$@  E
    6(   j6$  )$7  A,j ý( ! 
AG
  AÿÿqAG
  A	M
@ Axj   	  A
j-  At  Aj-  r"  Aj/  " At  AvrAÿÿq" jI
 B 7$@  E
    6(   j6$  )$7 A,j Ajý( " E
A !@  ("E
  Aq
   6    Aj6  )7  Aj¨6$ A$jš! ($! A 6$ E
  e  ^ AM
  Aj!  Atj! Aj" G
A ! A0j$   ½@  ("AM
  A|qAF
 @  ( "Aj(  " At  A€þqAtr  AvA€þq  Avrr"E
 A !@  AtAj" I
   kAM
@   j(  " At  A€þqAtr  AvA€þq  Avrr G
   Aj" G
 A     Ð	AÿÿÿÿqN# Aà k"$  AÀ 6  Aj6  )7    Ò	 AÜ j Ajü( !  Aà j$   ¬||@ 
   B€€€€x  B€€€€xU" Bÿÿÿÿ  BÿÿÿÿS§@@  ¹D     @@¢ Av¸  ¸£"D      àÁ D      àÁd"™D      àAcE
  ª!A€€€€x! AÿÿÿÿA  D      àÁf D  ÀÿÿÿßAe    6  ( A È  †# Ak"$   ( (! B€€€€€€À 7 B€€7   A È  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (   Aj$    8    6   6 A,ÝC¿" (Aj"6@ 
     6  J  (!  A 6@@ E
  ("E
  Aj"6 
   ( (     Î# Ak"$  A 6@ ("AL
 @@   (    AjÒ
 AÝC! (! A 6 Aà¦6  Ajƒ  6  6 Aì¨6   (Aj"6 
A ! E
  ("E
  Aj"6 
   ( (   Aj$      (- 	AqAv   (- AqAv   (- AqAv
   ((Aq   (- AqAv+# Ak"$  Aj  ((ü( !  Aj$   +# Ak"$  Aj  ((ü( !  Aj$   !~ (")4!   )<7   7 
   (/D
   (.F
   (.H    ((h) 7 # Ak"$   ("6  (! @@ E
    A  (  Ajç!A  A  ( F !   A A  Ajç!A  ( ! Aj$  
   ((š
}~# A k"$ @@ * C  €<”C  €G”"‹C   O]E
  ¨!	A€€€€x!	  	6@@ *C  €<”C  €G”"‹C   O]E
  ¨!
A€€€€x!
  
6@@ *C  €<”C  €G”"‹C   O]E
  ¨!A€€€€x!  6@@ *C  €<”C  €G”"‹C   O]E
  ¨!A€€€€x!  6@@@ ("
 A !@@@@  - q"AG
 Aq!
 - E
 ("
E
 
AjAL
 AF!
@A  
k"
AM
 AF!
 
Aô¨j,  !
@ - 0AG
   
 lAä m j6  
 	lAœm 
j6 - AG
  !
@ ( "	E
  	 	(Aj"
6 
E
 (!
 	   
(î	 	("E
 	 Aj"6 
  	 	( (      (Aj"6 E
  Aj   AjÚ	!@@  ("  AˆAŠ  (Aq"	¦E
 A ! 	Aq
   A
¦
  (T! @ As Eq
  - 
  AA j( "A‘H
 A ! Að|j"AçK
  Aì j (" Au"s k (" Au"s kj¬Aœ§A’© - A€F AÿÿqA
nj1  ~"§A¯žmA  B€€€€|B€€€€Tö‡(( A½A !   Ë
   (P"A€K
   (LA€K
 @AÝC  (d  (hÜ	"
(   (LAAˆ AF"ÀE
  
("( ! Aj Å (!  (X!	@ 
   - ^AG
 @  (L"
  
!AA AF!  (P!A !@A !
@ E
    lj!A !@   ljAA  	  (T lj Avj-   AqtA€qAv Ø6 Aj"  (P"
I
   (L! 
! Aj" I
  
!@  (L" l"E
  A  Ø6  (L!@ 
  
!   (T" Au"s k"  H!A !@@ E
    lj 	  (T lj Ö6  (L! Aj" I
  
! 
Ý	AâC Û	 A j$   ´# A k"$ @ Aj  ("Ë	"( E
 @ 
  A Í	A€€m!  6@@ 
   AÍ	A€€m6 AÎ	! AÏ	!  A€€m"6 A Aj°   A¦  (/D! (T((!	  A€€m"6 A Aj°   A¦ (T((Aèl  (/Dm" 	Aèl m" F
    k  kl   km j6 A Aj°  Ì	 A j$ ¯# AÀ k"$   ("A AÀ Ý A8j"A )”§7  A )Œ§70@ E
 @ ("E
 A€€˜~!	@ AjAJ
 A  k"AK
  Aô¨j,  At!	@@ E
  	Aä m! A0jAr! 	Aœm!  6  - AG
      (î	    (Aj"6@ E
  A,j   A0jÚ	!A !@  AA
  ((AˆÀ qAˆÀ F¦
 @ E
  - 
  (" A‘H
   Að|jA
n" Aã   Aã I! @@ - A€G
   Aœ§j-  AtA¯žn!   A€¨j-  !  (TAì j  ö B 7$ Aý6  Aþ6 Aÿ6 A€6 AÝC¢	"6 A€€€¬6 B 7 (TAì j Aj Ajò@ (  (F
  Ajô	 ¥	 ¤	AâCA ! Û	 AÀ j$   Ø}# Ak"$  ( !  ( !   (² *"•8  ² •8  AjA¨	 ( !  ( !  (² *"•8  ² •8   AjA¨	 ( !  ( !  (² *"•8  ² •8   AjA¨	  ( 6  (6 Aj$ A ž}# Ak"$  ( !  ( ! (!   ( ("kAtAm j² *"•8    kAtAmj² •8  AjA¨	 ( ! ( !  ( !  (  (" kAm  j² *"•8    kAmj² •8  AjA¨	 ( !  ( !  (² *"•8  ² •8   AjA¨	  ( 6  (6 Aj$ A h}# Ak"$  ( !  ( !   (² *"•8  ² •8  AjA ¨	   ( 6   (6 Aj$ A u}# Ak"$  ô	 ( ¥	 ( !  ( !   (² *"•8  ² •8  AjA¨	   ( 6   (6 Aj$ A ¨	}}@@  ( "("  ( "k"Am"AI
  !@  A~j"Alj"- AG
  ! - 	Aq
    *  j"Axj* [  *  Atj* [!@ AI
  A|j" O
  Alj"- AG
  - 	Aq
  A}j" O
  Alj"- AG
  - 	Aq
  *  * "\
  * *"	\
  A~j" O
  Alj"*  \
  * 	\
  Aj" O
    Alj"* 	[  *  [!@  M
    kõ	  O
 @  Alj"  F
 @   Atj¡	" G
   6 £@  ("  ("kAm I
 @ Al"E
   j!@ ž	Aj" G
  !   6@@   ( "kAm" j"AÖªÕªO
 @@  kAm"At"   KAÕªÕª AªÕªÕ I"
 A ! AÖªÕªO
 AlÝC! Al!  Alj"!@ Al"E
   j! !@ ž	Aj" G
   ( !  (!  j!@  F
 @ Atj Atj" 	!  G
   (!  ( !   6   6   (!   6@  F
 @  Atj¡	"G
 @ E
    kâCÅ6 ð n @ E
  - AG
      î	A !@  (" A¦
  (T(("Aäöü~jAÉíù}I
 @  (/D"
   Aèl m! L# Ak"$  A A€Ø6!  (  A€å A : ÿ AŒj ü( !  Aj$      ( Ì   ( äì# A k"$   B 7   AjB 7 @@ ("- 	A qE
   A€¦
 (T Aj¨ 
 (A Aj©  ("A›‰ƒ A›‰ƒH"Aåöü~ Aåöü~J! ("A›‰ƒ A›‰ƒH"Aåöü~ Aåöü~J! ("A›‰ƒ A›‰ƒH"Aåöü~ Aåöü~J! ("A›‰ƒ A›‰ƒH"Aåöü~ Aåöü~J!@ (X"	/"E
  	/"	E
  Aèl 	m! Aèl m! Aèl 	m! Aèl m!   6   6     (".F"  H6    .H"  J6 (§   A¦
  ("(T"($! ( "¬ /D"Ù	!
 ¬ Ù	! ("AuAÿÿÿÿs  j"	 	 s 	 sqA H¬ Ù	!   ("AuA€€€€xs  k"  s  sqA H¬ Ù	6   6   
6 @ AýÀŸðJ
    AÀ m j6  Aÿÿÿÿ6 A j$ µ ("(T"($! ( "¬ /D"Ù	! ¬ Ù	! ("AuAÿÿÿÿs  j"  s  sqA H¬ Ù	!   ("AuA€€€€xs  k"  s  sqA H¬ Ù	6   6   6   6 ú
~# Ak"$   ( AjAr"â"6@@@@  M
   A 6  B 7  )!  AÝC"6    Aj"6  7    6  ( § ã"6  K
  (E
 @@@   ("O
   )7  Aj!   ( "	kAu"
Aj"A€€€€O
@@  	k"Au"   KAÿÿÿÿ AøÿÿÿI"
 A ! A€€€€O
 AtÝC!  
Atj"
 )7   Atj! 
!@@  	G
  
!@ Axj" Axj") 7   	G
   (!  ( !	 
Aj!   6   6  	E
  	  	kâC   6  ( Axj(  ã"6  K
 (
  Aj$ Å6 ð Ü@ ((\"
   A :    A : @@@@@@@@@@@@ ("AàÐ½ÓJ
 @ AíÚÉ‹J
 A !@ A½ûîõ{j
  E
 AÂž‘ŠG
A! AîÚÉ‹F
 AµÎ¥“F
 A Àˆ»G
	A!@ AáÚå›J
 @ AÏ—úœyj  AáÐ½ÓF
 AóÒ©›G
	A
! AâÚå›F
 AãÒ¹«F
 AóÜ…»G
A
!
A!	A!A!A!A!A	!A!A! A!   6   A: 9@  (" (("E
   ($" AL
    O
   Atj( / ) @@  (" ((
 A !   ($" AJ
    9@  (" (("E
   ($" AL
    O
   Atj( /
 º@  (" (("E
   ($" AL
    O
 @@@@@@@@@@@  Atj( ("AàÐ½ÓJ
 @ AíÚÉ‹J
 A ! @ A½ûîõ{j
  E
 AÂž‘ŠG
A AîÚÉ‹F
 AµÎ¥“F
 A Àˆ»G
A@ AáÚå›J
 @ AÏ—úœyj  AáÐ½ÓF
 AóÒ©›G
A
 AâÚå›F
 AãÒ¹«F
 AóÜ…»G
A
AAAAAA	AAA!    >@  (" (("E
   ($"AL
   O
     Atj( ß    ( AtAø©j( ÞE   (  ÝEo  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    (!  A 6@ E
  Ñ  Aj„   t  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    (!  A 6@ E
  Ñ  Aj„  AâC 2  Ajˆ
  (!  A 6@ E
  J  Aj„  «@  (|"E
  Aj  Aø j‡@  (t"E
  Aj  Að j‡@  (l"E
  Aj  Aè j‡@  (d"E
  Aj  Aà j‡@  (\"E
  Aj  AØ j‡@  (T"E
  Aj  AÐ j‡@  (L"E
  Aj  AÈ j‡@  (D"E
  Aj  AÀ j‡@  (<"E
  Aj  A8j‡@  (4"E
  Aj  A0j‡@  (,"E
  Aj  A(j‡@  ($"E
  Aj  A j‡@  ("E
  Aj  Aj‡@  ("E
  Aj  Aj‡@  ("E
  Aj  Aj‡@  ("E
  Aj  ‡  8  Ajˆ
  (!  A 6@ E
  J  Aj„  AœâCP @ AO
    Atj" Aj!@  A j( " E
   Aj ‡  6@ E
  Aj †  @ AI
     AtjA j( ä# Ak"$  A 6 Aj¬    (6 Aô ÝC  ¥
!  B 7   6  B 7    Aj6    Aj6A!@  ( A½AG
   (  Aj Aj AjðA! ("AJ
  (!@ AG
  AJ
A ! AG
  AG
  (A J!   :   Aj$   ]  Aj  (Ž
  Aj  (
  (!  A 6@ E
  ¦
Aô âC  ( !  A 6 @ E
  ­   ? @ E
    ( Ž
   (Ž
@ (" E
   Aj Aj‡ A âCY @ E
    ( 
   (
@ ( " E
   Aj Aj‡ (!  A 6@  E
   ^ A$âC§# Ak"$   ( "6  Aj!@ E
   ( Aj6   :   6  Aj‘
! (!A ! A 6@ E
  ^@   AjF
  ( "E
   (Aj" 6 !  
   Aj$  á  Aj!@@  (" E
  !@@@  Aj" 
 @  E
   !@  (" ("G
   -  - I
  !  H
   !  Aj!   ( " 
   F
   Aj" 
    
@ ("  ("G
  -  - I
   N
 ! Ð# A k"$  “
!  ( "6  Aj! @ E
   ( Aj6   :   6  Aj6 Aj   AjAý$ Aj Aj”
 ("Aj! @ ( "E
  Aj  ‡  6 @ E
  Aj  † (! A 6@ E
  ^ A j$  å~AœÝC!  ) !  B 7  A 6 Aà¦6  Ajƒ A 6˜ AàÀ"6” A 6 AàÀ"6Œ A 6ˆ AàÀ"6„ A 6€ AàÀ"6| A 6x AàÀ"6t A 6p AàÀ"6l A 6h AàÀ"6d A 6` AàÀ"6\ A 6X AàÀ"6T A 6P AàÀ"6L A 6H AàÀ"6D A 6@ AàÀ"6< A 68 AàÀ"64 A 60 AàÀ"6, A 6( AàÀ"6$ A 6  AàÀ"6  7 AÐ¿"6   (Aj" 6@  
   þ# Ak"$ A !@  Aj Ÿ
"( "
 A$ÝC! ( ! A 6 ( !	 A 6  (!  	6@ E
  ^  (6 - ! A 6  AôÀ"6  :   (6 B 7   6  !@ ( ( "E
   6  ( ! ( ÈA!  (Aj6   :    6  Aj$ §@@  ("
 A !  Aj"! @    ( I (" I  F"!  AA  j( "
 A !   F
    (O   ("O  FAG
   (" E
     (Aj"6  ! 
   ¼ “
!@@@  ("
   Aj"!@@  "("I  ("I  F"AG
  ! ( "
@  I  I AF
  ! ("
  Aj!A ÝC"A 6 AôÀ"6  6 B 7   ­B † ­„7  6  !@  (( "E
    6 ( !  ( È    (Aj6 Aj!@ ("E
  Aj ‡  6@ E
  Aj † Œ~# Ak"$   ( !   ) "7   7A !@@     Þ	" E
 @  AÀ AÀ „
E
   !  ("E
   Aj"6 
     ( (   Aj$       AtAØ¿"j) 7    A )ÈÀ"7    A )ÐÀ"7 @  ("E
  Aj  ‡  "@  ("E
  Aj  ‡  AâC	   A 6@  ("E
  Aj  ‡  Ý  Aj!@  ("
   6  @@@@  " Aj"
   
@ ("  ("G
  -   - I
  N
  !  ( "
@  
   
@  (" ("G
   -  - I
  N
  Aj!  ("
    6  "@  ("E
  Aj  ‡  AâC	   A 6#   B 7  A :   A 6   AjB 7       ( !  A 6 @ E
  ^       (AtAm6  A­Ç;   B 7  A :     6  AjB 7   AjA 6   AjA AØ Ø6  é  (p!  A 6p@@ E
  ("E
  Aj"6 
   ( (    (l!  A 6l@ E
  ("E
  Aj"6 
   ( (  Aì !@   A|j"j"( ! A 6 @ E
  ("E
  Aj"6 
   ( (   A4G
 @  (("E
  !@   (,"F
 @ A|j"( ! A 6 @ E
  ^ Axj"( ! A 6 @ E
  ^  G
   ((!   6,   (0 kâC@  ("E
  !@   ( "F
 @ A|j"( ! A 6 @ E
  ^  G
   (!   6    ($ kâC  (!  B 7@ E
   ( (  @  ("E
  !@   ("F
 @ Axj"( ! A 6 @ E
  ^  G
   (!   6   ( kâC  (!  A 6@ E
  ^   3@ E
   A :    (!   6 E
   ( (    (!  A 6 ò# A0k"$   (! B 7( ( (! B 7@@@  AåÚ…ó Aj  "
 A ! AL
A ! AVA  Ø6!  (!   6$  6   ( (!  ) 7@   AåÚ…ó Aj   G
   6  6  )7  AÕ	! E
  J A0j$  Å6 ö~~# Ak"$ @  (E
   ( "6@ E
   ( Aj6   6@@  ("  (O
  A 6  (! A 6 ( !  6 @ E
  ^  (6 Aj!  Aj Aj«
!   6 (!A ! A 6@ E
  ^  Aj!B !B !@ ( "E
 A  Aj 5"P!@ ( "E
  ("	­B † AjA  	­„!@  B ˆR
   § §°7E
A ! Aj"	A  !@ E
  	 (j!@  F
 @@ -  AO
 Aj" F
  @  ("  ( ( "
   ("A A AA   ( (
 "E
  (!    ©
"6@ E
 @@@ (E
   (,"	  (0O
 	 6   ( Aj6  	 ( "
6@ 
E
  
 
( Aj6    	Aj6, A 6  A(j Aj ¬
!	 (!   	6, A 6 E
 ^   ( (  @@  ( "  ($O
   ( "6 @ E
   ( Aj6  Aj!  Aj Š!   6   ƒ Aj$ Ž@@  (  ( "kAu"Aj"A€€€€O
 A !@  ( k"Au"   KAÿÿÿÿ AøÿÿÿI"E
  A€€€€O
 AtÝC! ( ! A 6   Atj" 6   (6  Atj! Aj!@@  ("  ( "G
  !@ Axj"( ! A 6  Axj" 6  A|j A|j( 6  ! !  G
   (!  ( !   6   6   (!   6@  F
 @ Axj"( ! A 6 @ E
  ^  G
 @ E
    kâC Å6 ð ã@@  (  ( "kAu"Aj"A€€€€O
 @@  ( k"Au"   KAÿÿÿÿ AøÿÿÿI"
 A ! A€€€€O
 AtÝC!  Atj" ( "6 @ E
   ( Aj6  At!  ( "6@ E
   ( Aj6   j! Aj!@@  ("  ( "G
  !@ Axj"( ! A 6  Axj" 6  A|j"( ! A 6  A|j 6  ! !  G
   (!  ( !   6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ^ Axj"( ! A 6 @ E
  ^  G
 @ E
    kâC Å6 ð .@  ("E
   -  
     ( (   A:  ï	~~@  ("E
   -  
     ( (   A:    (!  ( !@@@@  F
@ A|j"( "E
   ( Aj6 B !@@ ¯
"
 A !B !A  Aj 5"P!@ ( "E
  ("	­B † AjA  	­„!A !@  B ˆR
   § §°7E!@ E
  ^ E
  ( " 
A   ((!
  (,!A ! @ " 
F
@ Axj"( "E
   ( Aj6 B !@@ ¯
"
 A !	B !A  Aj 5"P!	@ ( "E
  ("­B † AjA  ­„!A !@  B ˆR
  	 § §°7E!@ E
  ^ E
  A|j( " 
 A     ( Aj6   ¢# Ak"$    6 AjA n AjA-n AjA,n Aj AjA+A q@ - AG
  (" E
   Aj  6  Aj „ ( !  A 6   E
   ^ Aj’ (!  Aj$   Ã# A0k"$ @@@ A
J
 @   AtjA4j"( "
   (!  A(j ˜
  )(7 A !  A  A —
! ( !   6 @  E
   ("E
   Aj"6@ 
     ( (   ( ! E
  (Aj" 6 !  
  6 A: @ E
   6@ AqE
  ¤
@  (p"
   (! A jš
  ) 7A ! A  AjA —
!  (p!   6p@ E
  ("E
  Aj"6@ 
   ( (    (p! E
  (Aj" 6 !  E
 A¡–@  (l"
   (! Aj™
  )7A ! A  AjA —
!  (l!   6l@ E
  ("E
  Aj"6@ 
   ( (    (l! E
  (Aj" 6 !  E
 A0j$   Ã# A0k"$   6,  : +  ("  A,j ( ( @ AG
   ("  A+j ( (   (! B 7  ( (!	 B 7  AæÆÑ£ Aj 	 !  (!
 B 7 
( (! B 7A !	@  
 A  Aj  "
rE
 @@ E
      
²
! @ (,"E
   ( Aj6        
³
! A !	  E
   A,jƒ  - +: @ A¼A  ã	F
   6@ E
   â	
 @@ 
 At!A    Au"s kAH!  6  !	   ( (   (,! A 6,@ E
  ^ A0j$  	…# AÐk"$ @  I
   (!  AÐ j6H A€6L ( (!  )H7   AæÆÑ£ A j   AÐ jA€j! AÐ j!A !@ Aj(  Aj(  Aj(  Aj(  Aj(  Aj(  Aj(  (  jjjjjjjj! A j" G
 @@@@@  (  •
"
 @@ 
 A ! AV!  (!  6@  6D ( (!  )@7  AæÆÑ£ Aj   G
  (!  6<  68    A8j–
! (8! A 68 E
  J  (K
  (60  64  )07  Aj  kÖ	"‹
"E
  (Aj"6 
A ! E
 J  (!   (Aj"6 E
  (K
 (!  6,  6(  )(7@    Aj —
"
 A !   Š
 ("E
  Aj"6 
   ( (   AÐj$   ³# A0k"$   6,@@@@@@  ( A,j  
"
 A !@ E
  AV!  (!  6$  6( ( (!  )$7  A  Aj   G
  (!  6   6  A,j   Aj’
! (! A 6 E
  J A ‹
"E
  (Aj"6 
A ! E
 J  (!   (Aj"6 E
  (K
 (!  6  6  )7 A !    A —
"E
  A  Š
 ! ("E
  Aj"6 
   ( (   (,! A 6,@ E
  ^ A0j$   Ý'~~# Ak"$   ( "	6€@ 	E
  	 	( Aj6  At!	@@ E
  ( "E
  (E
  - AÿqAÀ G
  A€jA Ao A€jA n A€€ q! A ! 	Au!
@ (€"	E
  	(AI
  	Aj-  A+G
 @@@@ A€jA"E
 @ ("E
  Aj!
A !	@ 
 	j,  A¿jAO
 	Aj"	 G
  ^ (€"	
Ay!	 ^ 	(Ayj!	  A€j 	‘6ˆ A€j Aˆj„ (ˆ!	 A 6ˆ 	E
  	^ A ! 
 q!
 (€"	AjA²œ 	!
AÙ !AàÁ"!	@ 	 Av"Atj"Aj 	 (  
A H"!	  Asj  "
 @ 	A¨Ç"F
  	(  (€"AjA²œ 
  A€j 	- AtA€Á"j(   (€6| Aü jAç´‰!	@@@ 
  	E
  Aà´ A: @  (d"	
   (!	 AˆjA˜
  )ˆ7`A ! 	A  Aà jA —
!	  (d!   	6d@ E
  ("	E
  	Aj"	6@ 	
   ( (    (d!	 	E
 	 	(Aj"6 	! 
@ Aü jA†”‰E
  A“” A: @  (h"	
   (!	 AˆjA
˜
  )ˆ7 A ! 	A  A —
!	  (h!   	6h@ E
  ("	E
  	Aj"	6@ 	
   ( (    (h!	 	E
 	 	(Aj"6 	! E
 A 6x Aá˜6l A 6t A6p  )l7X Aˆj Aü j AØ jA p@@@@@@ - Œ"AG
   Aü j (ˆ6€ Aø j A€j„ (€!	 A 6€@ 	E
  	^ (x"	AjA²œ 	!
AÙ !AàÁ"!	@ 	 Av"Atj"Aj 	 (  
A H"!	  Asj  "
 @ 	A¨Ç"F
  	(  (x"AjA²œ 
  Aø j 	- AtA€Á"j( @@ (|"	
 A !	 	(!	 - ŒE
  Aü j 	 (ˆAsj‘6€ Aô j A€j„ (€!	 A 6€ 	E
 	^ Aø j Aü jƒA!@@@@ Aø jAÔ¥‰E
 A !A!A!@ Aø jA€à‰E
 A!@ Aø jA£É‰E
 A!@ Aø jAäÉ‰E
 A!A !@ Aø jAãò‰E
 A!@ Aø jA»à‰E
 A!@ Aø jAÎÉ‰E
 A!@ Aø jA‡Ê‰E
 A!A!@ Aø jAâ±‰E
 A!@ Aø jAõß‰E
 A	!@ Aø jA÷é‰E
 A
! Aø jAœì‰E
A! Aq"	AjAIAt" AÀ r 	AI!@@ 
  Aˆj Aø jA-r - ŒAG
 A !	@ (x"E
  (!	  Aø j 	 (ˆAsj‘6€ Aô j A€j„ (€!	 A 6€@ 	E
  	^ - ŒE
  Aø j (ˆ6€ Aø j A€j„ (€!	A ! A 6€@ 	E
  	^A !A !A! (x"E
  ("	E
  Aj!@@@ 	AI
   	jAyjAÁ¦A°7
A°Ç"! 	AI
@  	j"A}jAºÆA°7
 A¼Ç"!@@@@ 	A
I
  AvjAïëA
°7
AÈÇ"! 	AI
 AzjAåîA°7
AÔÇ"! 	AI
 A|jAùáA°7
AàÇ"!  Aø j 	 (k6ˆ Aø j Aˆj„ (ˆ!	 A 6ˆ@ 	E
  	^ (! AtAÐ q Aqr!A!A¼  A€€q!A !
@ (t"	E
  	(E
 A !
A !	A!@ (t"E
 (" 	M
  	jAj"A,  	k"¯7!@@ 
  B 7ˆ@  k  "Aj I
  B 7ˆ  6Œ  6ˆ  )ˆ7PA !A!@ A€j AÐ jý( "E
  ("E
  Aj!@@@ AI
  AÁ¦A°7
A°Ç"! AI
@ AºÆA°7
 A¼Ç"!@@@@ A
I
  AïëA
°7
AÈÇ"! AI
 AåîA°7
AÔÇ"! AI
 AùáA°7
AàÇ"!A !@@@@@ 	E
 A! 
Aq
A !
 E
A!A !	 
@ ("A€€qE
 A„A¼ A€€q! AÀ€q r!A ! AÀ qE
 A!
@ Aq
 A !A! AÀ r!A!A!
A!A!
A!@ E
  (Aj!  	j!	A !@ E
  ^ E
  AF
  Aø j Aü jƒA!@  (
      
  °
!@@ E
  ‚! AFAtA  AqAv! „! A¹‰6€ A6„  )€7H Aˆj Aø j AÈ jA p@@@ - ŒAG
 @ A€€qE
 AÅß!	 A6„ AÜ6€  )€7( Aˆj Aø j A(jA p@ - ŒE
 AÞ÷!	 A6„ A­Ä6€  )€7  Aˆj Aø j A jA p@ - ŒE
 AÏ÷!	 A	6„ AÆÒ6€  )€7 Aˆj Aø j AjA p - ŒE
A¥‰!	 A6„ A•ª6€  )€7@ Aˆj Aø j AÀ jA p@@ - ŒE
 AðÇ"!	 A6„ A·²6€  )€78 Aˆj Aø j A8jA p@ - ŒE
 AøÇ"!	 A6„ Aþ÷6€  )€70 Aˆj Aø j A0jA p - ŒAG
A€È"!	 	("	E
 Aø j Aˆj 	ü"	„ 	( ! 	A 6  E
  ^@ (x"	E
  	 	( Aj6   	¯
"6ˆ    Aˆj®
"	6h@ E
  ^ (h!	 AÀ q!@@@ 	E
  	(
B !@@ (x"
 A !B !A  Aj 5"P!@ (|"E
  ("­B † AjA  ­„!A !@  B ˆR
   § §°7E!A!@ 
  
   
AsrAqE
 @ E
   ( Aj6   ¯
"	6€    A€j®
6ˆ Aè j Aˆj„ (ˆ! A 6ˆ@ E
  ^@ 	E
  	^ (h!	 	E
 	(E! Av!	@@ AH
  E
 @ 
  Aø jA¢ª‰! A6„ AÈ"6€  )€7   ! Aˆj Aü j AjA p  
 A G!	 Aï q  !@@ - ŒAG
  (ˆ
 A	6„ AšÈ"6€  )€7 Aˆj Aü j AjA p - ŒAG
 (ˆE
 Aø jAÀÁ" A:  E
  6A !	 E
A!	 A:  A !A !
@@ (h"
  A
L
 (!@ AH
  E
@ E
  Aø j Aè jƒ A
J
@ E
  Aq
 @ A€€qE
 @ E
  Ar! Ar! Ar  ! Aø j AtA€Á"j( t  AÀ qAv 	r!	@@  ("  	Aq   Aø j ( (
 "E
 @ (|"E
   ( Aj6       	Aq 
  ±
!   ! 
A G 	 !
@ (h"	E
  	(E
 @  ("	 Aè j 	( ( "
      
  °
!@ (|"	E
  	 	( Aj6     	  
Aq 
  ±
!@@@  @ Aü jAç´‰E
  Aà´ A:   A  
  °
!   Aø j  A{q  
A  ´
!     
  °
!@  ("	  ("F
 @ 	( F
 	Aj"	 G
 @ 	 G
      
  °
!@  (" 	 ( ( "
 A !@ (|"	E
  	 	( Aj6     	  
Aq 
  ±
! (h!	 A 6h 	E
  	^ (t!	 A 6t@ 	E
  	^ (x!	 A 6x 	E
  	^ (|!	 A 6|@ 	E
  	^ Aj$   Æ ( "AjA²œ !AÙ !AàÁ"!@  Av"Atj"Aj  (  A H"!  Asj  "
 A !@@ A¨Ç"G
 A !A ! (  ( "AjA²œ 
   - "AtA€Á"j( A!   :    :  ‘~~@  ("  ( "F
  ) "B ˆ"§! §!@@@ ( " 
 B !  ("­B †  AjA  ­„!@ B ˆ R
  §  °7
 A Aj" G
 A ‘~~@  (("  (,"F
  ) "B ˆ"§! §!@@@ ( " 
 B !  ("­B †  AjA  ­„!@ B ˆ R
  §  °7
 A Aj" G
 A  Aü—A€Á"  ¹
A¸Á"G² @  ( ‰
 @  (‰E
  Aj@  (‰E
  Aj@  (‰E
  Aj@  (‰E
  Aj@  (‰E
  Aj@  (‰E
  Aj@  (‰E
  Aj@  ( ‰E
  A j@  ($‰E
  A$j@  ((‰E
  A(j@  (,‰E
  A,j@  (0‰E
  A0j A4j A8j  (4‰!    AþqAF   AI/   B 7  A¬È"6   B 7    Aj6  AjB 7   ‹  A 6  A¬È"6 @  ("E
  !@   ("F
 @ A|j"( ! A 6 @ E
  ^  G
   (!   6   ( kâC  Aj  (¾
  - @ E
    ( ¾
   (¾
 AjÏ
 AâC  A 6  A¬È"6 @  ("E
  !@   ("F
 @ A|j"( ! A 6 @ E
  ^  G
   (!   6   ( kâC  Aj  (¾
  A âCQ@  ("  (O
   ( "6 @ E
   ( Aj6    Aj6    Aj Š65   6@  ("  ("F
 @   Â
 Aj" G
 œ# Ak"$ @ Ã	"E
  A 6@  Aj Aj ( ( E
 @@@@ - AG
  AjAÀ˜‰
 AjA¿˜‰E
  AjA‘6 Aj’@@ AjAëÆ‰
  AjAè‰
 A ! AjAðÆ‰E
A! (! A 6@ E
  ^ E
  ( "6 @ E
   ( Aj6  Aƒ–…  Aj‡@@ - AG
    Â
   Ã
 ( ! A 6  E
  ^  Aj Aj ( ( 
  (! A 6@ E
  ^  ( (   Aj$ °
~~# Ak"$ @@ ( "AjA²œ Aˆðò6"E
  A A‚7 ˆ7! A A ‚7@ AA þ6AG
  ¬!@ ( "At A€þqAtr AvA€þq AvrrAæÆÑ£F
      A Ä
 - "A?K
 @@ - 	At Atr" - "r - 
At"r"­B†"	B€€€€ 	B€€€€T§"

 A ! 
AV!@@ A 
 þ6 
G
  E
   r r!A !@ At" 
K
  F
       j(  "At A€þqAtr AvA€þq AvrrÄ
 Aj" G
   E
 J é6 Aj$  Ý	# Ak"$ @  A ‚7A H
  Aà jAA þ6E
  - e! - d! A 6€ Aˆj A€j  Atr"At"j@ (ˆ A þ6
  (€! A 6€ E
 ^ A€j k (€"E
 @ (E
   Aj"	 AåÚ…ó Å
"
E
 @ 
("E
   6X  
Aj"6T  )T78  A8jAÕ	"6\ E
 @@ (
  A 6\  
(6L  6H  )H70  A0jAÕ	6P@ AÐ jAÁ¦‰
  A6Œ Aø£6ˆ@@ (P"
  B 7€ (! B 7€ E
   6„  Aj6€  )ˆ7(  )€7   Aü j A(j A jþ( 6D AÜ j AÄ j‡ (D! A 6D E
  ^@@  ("E
   Aj"!@   Aj AÜ j"
! AA  
j( "
   F
  AÜ j AjE
AÝC!@ ( "E
   ( Aj6 @ (\"E
   ( Aj6   ( Aj6   6 @ E
   ( Aj6  §!
  6@ E
   ( Aj6   6  ( Aj6  ^  
6  6@ E
  ^@ E
  ^A !A !@  	 A²ÞÌú Å
"E
 A ! (AÖ I
  AÚ j,  !
A !@ AÛ j-  "AqE
   ( AÜ jA€ª
A!@ AqE
   ( AÜ jA†ª
 Ar!@ AqE
   ( AÜ jAˆª
  Ar"6@ A(qE
   ( AÜ jAª
 A r! 
AJ
   ( AÜ jAª
 Ar!  ( AÜ jA ª
 A 6  Ar6 A6„ Aùá6€  )€7 Aˆj AÐ j AjA p@ - ŒAG
 A€€! A€€6 A6„ Aåî6€  )€7 Aˆj AÐ j AjA p@@ - Œ
  A6„ A‘Ê6€  )€7 Aˆj AÐ j AjA p - ŒAG
  AÀ r"6  Aj! A6„ A´Ç6€  )€7  Aˆj AÜ j A p@ - ŒAG
   Ar6  AÜ j6€ Aˆj  AÜ jAý$ A€j Aü jÆ
 (ˆ"(!  6@ E
  (! A 6@ E
  ^ (! A 6@ E
  ^ ( ! A 6 @ E
  ^ AâC E
  ^ (P! A 6P@ E
  ^ (\! A 6\ E
 ^ 
^ ^ Aj$ Æ# Ak"$ A !@ E
 @@  Atj"(  "At A€þqAtr AvA€þq Avrr F
 Aj" G
 A !A ! Aj(  "At A€þqAtr AvA€þq Avrr" Aj(  "At A€þqAtr AvA€þq Avrr"AsK
    j­S
 A !   A ‚7A H
 A ! A 6 Aj Aj j@ ( A  þ6E
  Aj k (! (! A 6 E
  ^ Aj$  ‰ Aj!@@@ ("
  !@@  "Aj"E
  ! ( "
@  E
  Aj! ("
A ! ( "
AÝC" ( ( "6@ E
   ( Aj6   6 B 7  A 6  6  !@ ( ( "E
   6  ( ! ( ÈA!  (Aj6   :    6 æ# Ak"$ @@@ AÔ¥‰E
 AÐÈ"!@ A€à‰E
 AØÈ"!@ A£É‰E
 AàÈ"!@ AäÉ‰E
 AèÈ"!@ Aãò‰E
 AðÈ"!@ A»à‰E
 AøÈ"!@ AÎÉ‰E
 A€É"!@ A‡Ê‰E
 AˆÉ"!@ Aâ±‰E
 AÉ"!@ Aõß‰E
 A˜É"!@ A÷é‰E
 A É"!A ! Aœì‰E
A¨É"!   Aj (ü"  ( ( ! ( !  A 6   E
   ^ Aj$  ›~# A k"$ A!@@@@@@@@ A€j	  A!A!A !A!A!A !A !	A !
A !@@ E
 @  ("E
   Aj"
!@   Aj "
! AA  
j( "
   
F
 A !
A !  Aj
 (!
@ AF
 A !
A ! 
( qE
 
!AA  A‘H 
("A€€qAvs"
 
Aj  AÀ qAvs"
 
Aj AqAv AqAvs"
 
Ar AÀ qAv AqAvs"
 
Aj  sAqAj"
AÄ F
A !
A !@@ ( "
 B ! AjA  ("	­!@@  ("  Aj"G
  !
 Aq! 	­B † „! A‘H! AqA G! AÀ qA G!@ (!@@@ AF
  ( qE
 Aj!A !@ E
 @@ ( "

 A !
 
(!
 	 
F!AA   ("
A€€qAvs" Aj  
AÀ qAvs" Aj  
AqAvs" Ar  
AqAvs" Aj  
sAq"
Aj 
 " 
  
J"
!   
!
 E
  
L
  7  7 Aj  AjA p - AG
 @ ( "E
  ( 	j" (O
  
! !
  jAj,  AŸjAI
 ! !
 
! !
@@ ("
E
 @ 
"( "

  @  ("( G!
 ! 

  
! !
 !  G
  

 @ 
  AqE
    AjAÀ‡ü"  ( ( !
 ( !
 A 6 @ 
E
  
^ 

A !
 A j$  
 A j@  ("
 A   Aj"! @    Aj "!  AA  j( "
 A !@   F
    Aj
   (! å
# Ak"$ A !@@ E
  A 6Œ@@@@ AæÆÑ£F
 @ 
 A ! (E
A ! ("E
 ("AI
 Av! Aj!	A !A !
A !@ (" At"I
  k"AM
@ 	 j"
(  "At A€þqAtr AvA€þq Avrr G
  AM
@ A|qAxj     
Aj(  !  
Aj(  "At A€þqAtr AvA€þq Avrr"6Œ At A€þqAtr AvA€þq Avrr!
 Aj" G
   (E
 (!  6ŒA !
@ 
 A ! ( I
 A ! ( "AjA²œ Aˆðò6"E
 A !@  
A ‚7A H
  AŒjO A A€Ø6"O@ ( "E
  ("E
  A€ A€I"Aj O
     kjAj Ö6 (  (ŒA þ6! (ŒA  AF! é6 Aj$     @ E
   Ajƒ A G A ‹  (!  A 6@ E
  (! A 6@ E
  ^ (! A 6@ E
  ^ ( ! A 6 @ E
  ^ AâC  ( !  A 6 @ E
  ^AÝC" A¸É"6      	   AâC á# Ak"$ A ÝC"B 7  AjB 7  AjB 7  AjB 7  ¼
"AÐÉ"6 @@‡("E
  ( "E
@  Aj ü"À
 ( ! A 6 @ E
  ^ Aj"( "
    AjAƒ“ü"À
 ( ! A 6 @ E
  ^  AjAý’ü"À
 ( ! A 6 @ E
  ^  AjA‡„ü"À
 ( ! A 6 @ E
  ^  AjA”“ü"À
 ( ! A 6  E
  ^ Aj$     ½
A âCÄ
# AÐ k"$ @   Ç
"
 A!@@@@@@@ A€j	   Aj! AÈ jA”²ü!	@@  ("E
  !
@ 
  Aj 	"!
 AA  j( "
  
 F
  	 
AjE
 !
 	( !A ! 	A 6 @ E
  ^ 
 F
 
(!  Aj! AÈ jAÇü!	@@  ("E
  !
@ 
  Aj 	"!
 AA  j( "
  
 F
  	 
AjE
 !
 	( ! 	A 6 @ E
  ^ 
 F
  Aj! AÈ jAÜü!	@@  ("E
  !
@ 
  Aj 	"!
 AA  j( "
  
 F
  	 
AjE
 !
 	( ! 	A 6 @ E
  ^@ 
 G
  	A‹Àü!	@@ ( "E
  !
@ 
  Aj 	"!
 AA  j( "
  
 F
  	 
AjE
 !
 	( ! 	A 6 @ E
  ^ 
 G
  	AÒþü!	@@ ( "E
  !
@ 
  Aj 	"!
 AA  j( "
  
 F
  	 
AjE
 !
 	( !A ! 	A 6 @ E
  ^ 
 F
 
(! A6D AÊï6@  )@78A ! AÈ j  A8jA p@@@ - L
  A6D AÆþ6@  )@70 AÈ j  A0jA p - LAG
 A6D Ažï6@  )@7 AÈ j  AjA p - L
 A
6D AÄþ6@  )@7  AÈ j  A p - LAs! A6D A’«6@  )@7( AÈ j  A(jA p@@ - L
  A6D A¯€6@  )@7  AÈ j  A jA p - LAG
 A6D Aðª6@  )@7 AÈ j  AjA pA! - L
 A6D A­€6@  )@7 AÈ j  AjA pAA - L!A AqAv A‘H!  Aj! Al"AˆÊ"j! AôÉ"j!
@@ AÈ j 
( ü!	 !
@@ ( "E
 @ 
  Aj 	"!
 AA  j( "
  
 F
  	 
AjE
 !
 	( ! 	A 6 @ E
  ^ 
 G
 
Aj"
 G
 A ! 
(! 	A‹Àü!	@@ ( "E
  !
@ 
  Aj 	"!
 AA  j( "
  
 F
  	 
AjE
 !
 	( ! 	A 6 @ E
  ^ 
 G
 	AÜóü!	@@ ( "E
  !
@ 
  Aj 	"!
 AA  j( "
  
 F
  	 
AjE
 !
 	( !A ! 	A 6 @ E
  ^ 
 G
        È
! 
(! AÐ j$     A6  A€Ë"6 É# Ak"$ AàÊ"!@@@@@@@@@  A±J
 @  A€j						 A€Ë"!  E
  A²F
  AÌF
  AîG
A°Ë"!AˆË"!AË"!A˜Ë"!A Ë"!A¨Ë"!A¸Ë"! (! Aj ü( !  Aj$   ÿ@@  Aÿ O
 A !A†!  A€@jAÿÿqAð I
   AÀÿqA€à F
   A€ä~jAÿÿqA¦£I
   A¹0jAÿÿqA-I
 A€!  A€jAÿÿqAðI
   AÀŸjAÿÿqAÀI
   AðÿqAðã F
 A!  AÐjAÿÿqAà I
   A€¨jAÿÿqA°× I
   A€þq"A€"F
 @  A€ÿqA€G
 AÞ!A¡!  AyjAÿÿqAI
  A€>F
 A²! A€F
   A°	jAÿÿqA­I
 @  AðtjAÿÿqAð O
 A±!@ A€G
 AÌ!@  A€~jAÿÿqAÐO
 Aî!A£A  A€<F! AÿqA   A : 0  B 7(  B 7   AjB 7   AjB 7   AjB 7   A jA 6    @  (" 
 A   (ƒ  B 7  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (  @  ("E
    6 J  (!  A 6@ E
  £
AâC  (!  A 6@ E
  ("E
  Aj"6 
   ( (    ( !  A 6 @ E
  ("E
  Aj"6 
   ( (     Õ# Ak"$   B 7(   : 0AÝC¢
!	  (!   	6@ E
  £
AâC‡((        (´
!  ( !   6 @@ E
  ("E
  Aj"6@ 
   ( (    ( !@ E
  Aj ê	   )7 Aj$  
    A A ß
·@@  ("
 ‡(  !  (!   6 E
  ("E
  Aj"6@ 
   ( (    (!  (Aj"6 E
       ü
!  ("E
   Aj"6@ 
   ( (     #@  ( "
 A       (ö	©# Ak"$    7(   : 0@@@@ ("
 A !A !A ! AL
 ( ! AV!@@ Aq"	
  ! !A ! ! !@  -  :   Aj! Aj! Aj" 	G
 @ AI
   j!@  -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj" G
   j!@  ("E
    6 J   6   6   6‡(!  (!   ("6   k6  )7  A  A —
!  ( !   6 @ E
  ("E
  Aj"6 
   ( (      ("6    ( k6  ( ! Aj$  A GÅ6  @  ( " 
 A   à	$@  ( "
 A  è	¬  ( ç	Ù	$@  ( "
 A  é	¬  ( ç	Ù	ž# A k"$ @@  ( "
 A ! @ â	E
 A!    ( å	6 Aj’ A6 Aœè6  )7  Aj Aj A p (! A 6 - !  E
  ^ A j$   Aq @  ( " 
 A   ã	 @  ( " 
 A   á	b# Ak"$ @@  ( " 
 A ! @ Aj  (æü"( " E
   (
 AÄÊ" ( !  Aj$   D@  ( "
 A !@  (" E
   ( " E
     ( Aj6   !  ä	¤# Ak"$ @@@@  ( "
   (" E
  ( " E
    ( Aj6  ä	" 
 AjAÄÊ"ü@@  (E
    6    ( Aj6  AjAÄÊ"ü  ^ (!  Aj$   §# A0k"$ @@@@  ( "
  A 6@@@ A(j (æü"( "E
  (E
   6 AÄÊ"  ( "6 E
 (E
 AjAÄÊ"‰
   (6  ( "E
   å	"6   ê
6@  ( "E
  à	E
  AjA n E
@ (E
  AjAÁ¦‰
 @@  ( " 
 Aø£! Aá˜Aø£  à	!  A6,   6(@@ (" 
  B 7   (! B 7  E
   6$   Aj6   )(7  ) 7  Aj Aj Aj þ" ‡  ( !  A 6  E
  ^ (!  A 6  E
  ^@  (" E
    ( " 6  E
    ( Aj6  A 6 (!  A 6  E
   ^ (!  A0j$   Ù}}}@ ( "
   A :   A :     æ	  A: @@ ( ç	"E
   - E
@@  ( ²C  zD” ³"•"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!   AÿÿÿÿA  C   Ï` CÿÿÿN_6 @@  (²C  zD” •"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!   AÿÿÿÿA  C   Ï` CÿÿÿN_6@@  (²C  zD” •"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!   AÿÿÿÿA  C   Ï` CÿÿÿN_6@@  (²C  zD” •"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!   AÿÿÿÿA  C   Ï` CÿÿÿN_6t &@  ( "
 A      - 0  (ï	½@@  ("
 ‡(  !  (!   6 E
  ("E
  Aj"6@ 
   ( (    (!  (Aj"6 E
          ø
!  ("E
   Aj"6@ 
   ( (     µ@@  ("
 ‡(  !  (!   6 E
  ("E
  Aj"6@ 
   ( (    (!  (Aj"6 E
      ö
!  ("E
   Aj"6@ 
   ( (     `   A 6  Aà¦6   Ajƒ  B 7   6  AÐË"6   B 7(    Aj6  B 74    A(j6$    A4j60  z  A0j  (4ò
  A$j  ((ó
  Aj  (ô
  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    Aj„   % @ E
    ( ò
   (ò
 A âCE @ E
    ( ó
   (ó
 ($!  A 6$@  E
   ¤	AâC A(âCL @ E
    ( ô
   (ô
 Aj (û
 (!  A 6@  E
   ^ A âC  A0j  (4ò
  A$j  ((ó
  Aj  (ô
  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    Aj„  A<âC ó# A0k"$ A !@  ("E
 @@@@ ("AF
 @@ 
   ( (   AG
 AF
@@ ("
 A !A !	A ! - 0! (! (!	  6  	6  6  6  Aq"
:   (("E
  A(j"!@@@ (" F
   I!@ (" F
   H!@ (" 	F
   	H!@ (" F
   H! -   
I!   ! AA  j( "
   F
@@  ("F
   I!@  ("F
   H!@ 	 ("F
  	 H!  ("F
  H! E
  -   
M
   í
!  Aj6$ A(j  A$j" AjAý$ A$j A#j÷
 (("($!  6$@ E
  ¤	AâC  Aj6$ A(j  AjAý$ A$j A#j÷
 ((! ($! A0j$  ¹@@@@ ("
  Aj"! (! (!	 (!
 ( ! - Aÿq!@@@@@@  "("G
 @ 
 ("G
 @ 	 ("G
   ("G
  -  I
 -   O
	 	 H
  	H
 
 H
  
N
  I
  O
  N
 ! ( "
  N
 ("
  Aj!A(ÝC" ( ") 7 A j Aj( 6  Aj Aj) 7  A 6$  6 B 7   6  !@ ( ( "E
   6  ( ! ( ÈA!  (Aj6A ! !   :    6 ·	}# A k"$ @@ AG
 A !@@ *C @F”"	‹C   O]E
  	¨!
A€€€€x!
@@ *C @F”"	‹C   O]E
  	¨!A€€€€x!@@ *C @F”"	‹C   O]E
  	¨!A€€€€x!@@ * C @F”"	‹C   O]E
  	¨!
A€€€€x!
@@ ("
  A4j!A! ! AÀ j! - 0!  )78  64A	!  60  
6,  6(  6$  
6   6   A j6  6  At6  )7     Aj Ajý"    ù
! ( ! A 6  E
  ^ A j$  ¥# A k"$ @@  ("	E
   Aj"
!@  	 	Aj "! 	AA  j( "	
   
F
   AjE
 B 7  Aj"
6  6 Aj  Aj" Aý$ Aj Ajú
 ("	Aj" 	(û
 	 (6 	 ("6 	 ("
6 	Aj!	@@ 

   	6   	6 B 7  
6A ! Aj û
  6 Aj  Aý$ Aj Ajú
 (! Aj!
@@ ("
E
  
! 
!	@  	 	( I"! 	 Atj( "	
   
F
   (I
  (!@@  ("	
 A ! 	      í	! 
( !
 
!	@@ 
E
 @@  
"	("O
  	!
 	( "

@  I
  	! 	("

  	Aj!
AÝC"A 6  6  	6 B 7  
 6  !	@ (( "E
   6 
( !	 ( 	È  (Aj6 (!	  6 	E
  	Ý	AâC A j$  “ Aj!@@@ ("
  !@@  "Aj"E
  ! ( "
@  E
  Aj! ("
A ! ( "
A ÝC" ( ( "6@ E
   ( Aj6  B 7  6 B 7   Aj6  6  !@ ( ( "E
   6  ( ! ( ÈA!  (Aj6   :    6 E @ E
    ( û
   (û
 (!  A 6@  E
   Ý	AâC AâCç# A k"$   6  6  6@@  (4"E
   A4j"!@@@ ("	 F
  	 I!	@ ("	 F
  	 H!	 ( H!	   	! AA  	j( "
   F
 @@@  ("F
   I!  ("F
  H! E
  (N
    à
!  Aj6 Aj  A0j"	 AjAý$ Aj Ajý
 ( 6  Aj6 Aj 	 AjAý$ Aj Ajý
 (! (! A j$  Ô@@@@ ("
  Aj"! (! (!	 ( !
@@@@@@ 
 "("G
  	 ("G
  ("H
  N
 
 I
  
I
 	 N
 ! ( "
  	N
 ("
  Aj!A ÝC" ( "
) 7 Aj 
Aj( 6  A 6  6 B 7   6  !@ ( ( "
E
   
6  ( ! ( ÈA!
  (Aj6A !
 !   
:    6 &   B 7  B 7    Aj6     Aj6     Aj  (€    (€  ? @ E
    ( €   (€@ (" E
   Aj Aj‡ AâC@@@ ( "
 A!  (Aj"6 E
A !   j"Aj!@@ ("E
  ! @    ( I"!   Atj( "
    F
    (I
   ("E
   (Aj"6 
A<ÝC!@ E
   (Aj" 6  E
  ð
" (Aj"6 E
 !@@ ( " E
 @@   "(" O
  ! ( " 
@   I
  !  (" 
  Aj!AÝC" A 6  AàË"6   6   6  B 7    6   !@ ( ( "E
   6  ( ! ( È  (Aj6  Aj!@  ("E
  Aj ‡   6 Aj †@ E
  ("E
  Aj"6 
   ( (    @  ("E
  Aj  ‡  "@  ("E
  Aj  ‡  AâC	   A 6vAÝC"Ð
"6  A$ÝCŒ
6AÝCþ
!   6  6A  6Ðƒl  ( (  A (Ðƒl"(( ( " ( ( §
ˆ@A (Ðƒl" E
   (!  A 6@ E
  ÿ
AâC  (!  A 6@ E
  
A$âC  ( !  A 6 @ E
   ( (    AâCA A 6Ðƒl	 A (Ðƒl:   B 7  B 7   A$jB 7   AjB 7   AjB 7   AjA ;   ^    ) 7   A(j A(j( 6   A j A j) 7   Aj Aj) 7   Aj Aj) 7   Aj Aj) 7      ó
}}}}}}}}}}C  €?!C    !C    !C  €?!@ - AG
  *(! *$! * ! *! *! *! * !	 *!
    *"” *" ”’8    	” 
 ”’8    ”  ”’8    	”  
”’8     C    ” C    ”’’8    	C    ” 
C    ”’’8†}}}}}}}C    !C    !C    !C    !@@@@  ( Aj  ²!  *"C     C  €?_C     C    `"! ! ²!  *!  *!  *!C    !@@  *"C    `E
 C    !C    ! C  €?_E
C    !  *"C    `E
 C    !C    ! C  €?_E
C    !  *"C    `E
 C    !C    ! C  €?_E
C    !  *"C    `E
 C    !C    ! C  €?_E
C  €?  ’"C  €? C  €?]“!C  €?  ’"C  €? C  €?]“!C  €?  ’"C  €? C  €?]“!C    !C    ! ²!@@ C  C”"‹C   O]E
  ¨! A€€€€x!   At! @@ C  C”"‹C   O]E
  ¨!A€€€€x! At  r! @@ C  C”"‹C   O]E
  ¨!A€€€€x!   r! @@ C  €O] C    `qE
  ©!A !   Atrà} ( !  B 7   6   AjB 7 @@@    A6   C    C  €? “" C    ]"8   8   8  C     * “" C    ]8  C     * “" C    ]8  C     * “" C    ]8  C     * “" C    ]8¾ ( !  B 7   6   AjB 7 @@@    A6   C  €? •"8   8   8   ) 7   Aj" Aj) 7   Aj" Aj( 6     * •8  *  •8     * •8  *  •8 C   B 7  AôË"6   B 7  AjB 7   AjB 7    A$jB 7   A,jA 6   Ï# Ak"$   AôË"6 @  (,"E
  A  ( (    (," ( (0   A$j Aj) 7    ) 7  (,!  A 6, E
   ( (    (!  A 6@@ E
  ("E
  Aj"6 
   ( (   Aj$    `# Ak"$ @  (,"E
    ( (    (," ( (0   A$j Aj) 7    ) 7 Aj$ Ô# Ak"$   AôË"6 @  (,"E
  A  ( (    (," ( (0   A$j Aj) 7    ) 7  (,!  A 6, E
   ( (    (!  A 6@@ E
  ("E
  Aj"6 
   ( (    A0âC Aj$  )    8  B 7   8     ’8   Œ8ä# Ak"$   (,!   6,@ E
   ( (    (,!   A ( ( 6    (,"A ( ( 6    (,"A ( ( 6    (,"A ( ( 6    (," ( ( :    (," ( (0   A$j Aj) 7    ) 7 Aj$    (,"   ( (     (,"    ( ( '@  (" E
     (Aj"6 
    F  (!   6@@ E
  (" E
   Aj" 6  
   ( (   (  (   AˆA A  (" Aq  AÀ qÀ   (,"    ( ( f# Ak"$ @  (,"    ( ( "E
    (," ( (0   A$j Aj) 7    ) 7 Aj$  f# Ak"$ @  (,"    ( (  "E
    (," ( (0   A$j Aj) 7    ) 7 Aj$  Ó# A k"$  Aj¢	" ( ² (² (² (²«	 A; @  (," A  Aj ( ( "E
  Aj  (," ( (0   A$j" Aj") 7    )7 Aj  (," ( (0   ) 7    )7 ¤	 A j$  Ô
}}}}}# A0k"$  AvA  -  "!	 ("
 ( "k"Am!
@@@ AvA  "
 @ AG
   ) 7  Aj) 7 @ E
  A(j  Aj˜	  )(7 A(j  ˜	  )(7    Aj   ŸA!@@@@ - Aq
  Aj  ´	 -  E
   Ajý óE
@@ *" *"“"‹C   O]E
  ¨!A€€€€x!@ A J
 A! ( " (G
 A! Aj" AsqA H
  6@@ *" *"“"‹C   O]E
  ¨!A€€€€x!@ A J
 A! (" (G
 A! Aj" AsqA H
  6@ (" ( "k AjH
 @  ²“ ² “^E
  Aj" AsqA H
  6  A€€€€xF
  Aj6@ (" ("k AjH
 @  ²“ ² “^E
  Aj" AsqA H
  6 A€€€€xF
  Aj6A!     
 E
 - AÐ q
  (," ( (P !A ! A 6 B 7 
 F
 A G!A !@@@@@  Alj"- Aj    Aj   - AqAv  	¡@ ("
 ("F
 @ 
 Atj¡	"G
   
6@ 
 (O
  
  	Aj! Aj ¢!@@  (O
    	Aj! Aj ¢!  6 Aj"
 
O
  
Alj!
@@  (O
   
 	Aj! Aj 
¢!  6 Aj" 
O
  Alj!
@  (O
   
 	Aj! Aj 
¢!@  (O
    	Aj! Aj ¢!  6 Aj" 
O
  A !    Aj  A G - AqAv  	¡ ("
E
  
!@ 
 ("F
 @ 
 Atj¡	"G
  (!  
6  ( kâC AÿF
  E
  	E
  - AqE
         £!  (,"       ( ($ ! A0j$  š# A0k"$ @@ A€€€xI
 A!  (,"    ( (, 
 Aj»	! Aj¢	" A¨	  A ¨	  (," A  A    ( ($ ! ¤	 ¼	 A0j$  †# Ak"$ A!@@  (,"   ( (( 
 A !  - AqE
 A,ÝC¿" (Aj"6 E
@  ( ( k ( (kAˆA A  ("Aq AÀ qÀE
   (,!  (Aj"6 E
   (  ( ( (4 E
  A A  ( ( k ( (k ØE
  ( ! (! (! (! B 7    k6   k6  (," A    A  ( (< A! ("E
  Aj"6 
   ( (   Aj$   à}}~}}}}}}# A0k"$ @ (  (F
  Aj¢	!@ ( ( "	k"Am"
AI
 @@@@@@@@ 
A~qAG
  	- AG
 	Aj-  
@@ A$F
  	*! 	* ! 	A j-  
 	* " 	Aj* \
 	*" 	Aj* \
@  	Aj* \
   	Aj* [
  	) "
7 @@@@ E
  
A!  A¨	  	Aj) 7  !
 A(j  ˜	 )(!
 	- !@@ 
§¾"‹C   O]E
  ¨!
A€€€€x!
  
²C   ?’8 @@ 
B ˆ§¾"‹C   O]E
  ¨!
A€€€€x!
  
²C   ?’8   Aÿq¨	  	Aj) "
7 @ E
  A(j  ˜	 )(!
@@ 
§¾"‹C   O]E
  ¨!A€€€€x!  ²C   ?’8 @@ 
B ˆ§¾"‹C   O]E
  ¨!A€€€€x!  ²C   ?’8A !
   	Aj-  ¨	 
AI
 
AqE
 ¢	! 
Av"Aj 
O
 Aj!A !@@  j" 
O
 	  kAlj"Atj"*  	 Alj"* \
 Axj*  *\
 A|j-  AF
 - AF
  A¨	  A ¨	 Aj" G
   A ¦	 ¤	 !
 ¤	 »	A 6A !@@@@ 	 Alj"- Aj  Aj! 	 Aj 
pAlj"- 
  Aj" 
O
 * !@@@ 	 Alj"* " * "\
   [
 *! *! *! *" *"“"  *"“"”C    ^E
     ‹ ‹]"A¨	    A ¨	@@  \
   \
   “"  “"”C    ^
  [
  [
  [
  [
  “  “”  “  “”\
  “!  “!    ‹ ‹]"A¨	    A ¨	 Aj" 
O
    ( ! (!	@ 
AI
  !
 	 G
 	 F
 »	A 6 !
 »	A 6 AtA€€€øq Aÿÿÿqr!A !@ E
 @ * C  €?\
  *C    \
  *C    \
  *C  €?\
  *C    \
 A ! *C    [
 
! A : ( A‚A€ : )  (,"
   A   A(j 
( ($  ¼	 ¤	 A0j$ ¯@@  (  ( "kAm"Aj"AÖªÕªO
 @@  ( kAm"At"   KAÕªÕª AªÕªÕ I"
 A ! AÖªÕªO
 AlÝC!  Alj!  Alj  	"Aj!@  ("  ( "F
 @ Atj Atj" 	!  G
   (!  ( !   6   6   (!   6@  F
 @  Atj¡	"G
 @ E
    kâC Å6 ð æ# A€k"$ A !@@  - AqE
  Að jAj"B 7  B 7p@@ E
  A0j  * *­	  A0jAj) 7   )07p A0j ¬	  A0jAj) 7   )07p@ E
  A0j  Að j•	 Að jAj A0jAj) 7   )07p Aà j Að jýA ! Aà jóE
 A,ÝC¿"	 	(Aj"
6 
E
A,ÝC¿"
 
(Aj"6 E
@@@ 	 (h (`k (l (dkAˆA A  ("Aq AÀ qÀE
 @@ 	- AqE
  	 	(Aj"6 
  (,! 	 	(Aj"6 E
  	 (` (d ( (4 E
 	 	(Aj"6 E
 
 	Â A0j¶! 	 	(Aj"6 E
  	 
A» B 7( B€€€€€€€À?7  B€€€ü7@ E
  AjAj Aj) 7  AjAj Aj) 7   ) 7 AjA  (`k²A  (dk²Ž	 (,"  Aj     ( ($ E
 B 7  (l (d"k6  (h (`"k6  (,"  	A  Aj  A   ( (< ! · 
(" E
 
  Aj" 6  
 
 
( (   · 	(" E
 	  Aj" 6  
  	 	( (   A€j$   _ @  - AqE
   (,"      ( (4 @@ E
  (" E
   Aj" 6  
   ( (  A        A ¦Ñ# A0k"$  (! (!  6$  6    j6,   j6( A j  Ajõ@@@@@@ ((" ( "L (,"	 ($"
Lr"
   	 k6   k6  
 k"6   k"6@@ E
   ("AqE
@ - AqE
   ("AqE
  (," A  Aj  
  ( (< ! AqE
 A,ÝC¿" (Aj"6 E
   k" 	 
k"
A ÀE
  (,!  (Aj"6 E
   (  ($ ( (4 E
A ! A A   
    A A ÕE
  
6  6 B 7   (," A   (  ($A  ( (< ! ! ! ("E
  Aj"6@ 
   ( (   
 A ! ("E
  Aj"6 
   ( (   A0j$      (,"   ( (8 G# Ak"$   )7 B 7   (,"      A   ( (< ! Aj$  ”# A0k"$  Aj™	!   j6,   j6(  6$  6  Aj  A$j) 7    )7 Aj A jõ  (,"        Aj A   ( (@! ! A0j$  "    (,"     A  ( (D8 "    (,"       ( (D8    (,"     ( (H “"~# Aðk"$  ( !  ) 7¸A!	A !
@@@ Aj"AK
   - 
 A!	A!A!
A !A !  (AH
@@@‡(-  
 µE
A! A6¸A !
A!	@  - AqE
 A !A!
A !
A!  (AH
 A!A!@ ( "E
  (E!  ( AGr!
A !A !
A !
A!A !
A !A!A !	A!A!
A !A !@@ - AG
   (,!  ) "7° ( (L!  70A!  A0j     A¸j U 
 A˜jAj Aj") 7  A˜jAj Aj") 7   ) 7˜ A€jAj ) 7  A€jAj ) 7   ) 7€ A˜j  Œ	@ *˜‹ *œ‹’C  HB^E
  ( "E
  (E
  A : P  AKAt: Q  ) "7(  7x   A(j   A A  A A  AÐ j®! (!A ! A 6t B 7l@@@@@@ 
 A ! AÍ™³æ O
  Al"ÝC"6p  6l   j"6t@ Ç	Aj" G
   6p (l! ) "B ˆ  kAm­V
@ A,l"E
  §" j!@ AÐ j A€j ˜	  )P"7 §¾!@@ E
  •! *!@@ Ž"‹C   O]E
  ¨!A€€€€x! B ˆ§¾!  6  •6 AÐ j  A˜j‹   ( -  AÐ j ( 	 A¸jî
6  Aj! A,j" G
 @ E
  (p" (l"kAm"AI
 @ Apj(  (F"
  Atj(  (G
 Aj"AI
 AA !AhAd !AA !@ !@ (l" "Alj" j( "  Alj" jAXj"( "k" s  sqA H
   j*   j* “‹ ²‹“C   ?_
 AA AH" j" s  sqA H
   6  Aj"AK
  AÀ j Aì j 	Ô	 AÀ j  AjõA! (H" (@"L
 (L" (D"L
A,ÝC¿" (Aj"6  k!  k!@ 
E
  E
@   AÀE
 @ (l" (p"F
 @@ ( E
   6ì  6è AÐ j  AèjÊ	 - XAG
  (T!	 (P! ( (" (Aj"6 E
   	 ( ( A A × Aj" G
  (D! (@!  )7X B 7P  (,"   AÐ j  A  ( (< ! ("E
  Aj"6A ! 
  ( (   E
@@  (AG
    AˆÀ
   AˆA A  ("Aq AÀ qÀE
@ - Aq
  AÉ  (Aj"6 E
    (@ (D¤E
A ! A 6<A !A !
A !	@ E
  AÐ j š	  (P"6< Aÿq!
 Av!	 AvAÿq! AvAÿq!@ (l" (p"F
 @@ ( E
   6ì  6è AÐ j  AèjÊ	 - XAG
  ( "("(! (!@ E
  (T! (P!  (Aj"6 E
       A A A A A ÖE
 (P" Am"j" s  sqA H!@@ *C  @@”"‹C   O]E
  ¨!A€€€€x! Ao! 
  A  A J"    H"N
  AH
  Aj! (T"As! A /"AvAq A€q"A	v"l!   kAl!!A !"@@ " j"A  qAJ"A H
   (N
  Aèj ( "Æ ! (ìK
 (è! Aèj  Æ (ì"  I
  !j! (è  j!   k! !@@@  @@@ 
E
  Aj-  !# Aj-  !$ -  !%  6ä  6à  )à7 A G # $ %jjAn A<j Aj¯ E
  Aj-  AüË"j-   	lAÿn"#Aÿs -  l 
 #ljAÿn:   Aj"# Aj-  AüË"j-   	lAÿn"$Aÿs #-  l  $ljAÿn:   Aj"# -  AüË"j-   	lAÿn"$Aÿs #-  l  $ljAÿn:   E
  AM
 AjAÿ:    I
  j!  k! Aj! Aj" G
  @@ 
E
 @@ AJ
  -   Aj-  j Aj-  j! Aj-   -  j!  6Ü  6Ø  )Ø7  A G An A<j A j°@@ AJ
  AM
 Aj" Aj-  AüË"j-   	lAÿn"#Aÿs -  l  #ljAÿn:   AM
 Aj" -  AüË"j-   	lAÿn"#Aÿs -  l  #ljAÿn:    Aj-  AüË"j-   	lAÿn"Aÿs -  l 
 ljAÿn:   E
  AM
	 AjAÿ:   !  I
@  F
  j!  k!@@ 
E
  Aj-  !# Aj-  !$ Aj-  !%  6Ô  6Ð  )Ð7 A G # $ %jjAn A<j Aj¯ E
  Aj-  AüË"j-   	lAÿn"#Aÿs -  l 
 #ljAÿn:   Aj"# Aj-  AüË"j-   	lAÿn"$Aÿs #-  l  $ljAÿn:   Aj"# Aj-  AüË"j-   	lAÿn"$Aÿs #-  l  $ljAÿn:   E
  AM
 AjAÿ:   Aj! Aj!  O
 	 @@ 
E
 @@ AJ
  Aj-   A~j-  j -  jAn! -  An!  6Ì  6È  )È7 A G  A<j Aj°@@ AJ
  AM
 Aj" A~j-  AüË"j-   	lAÿn"#Aÿs -  l  #ljAÿn:   Aj" Aj-  AüË"j-   	lAÿn"#Aÿs -  l  #ljAÿn:   E
	  -  AüË"j-   	lAÿn"Aÿs -  l 
 ljAÿn:   E
  AM
 AjAÿ:   !  I
@  F
  j!  k!@@ 
E
  Aj-  !# Aj-  !$ Aj-  !%  6Ä  6À  )À7  A G # $ %jjAn A<j ¯ E
	  Aj-  AüË"j-   	lAÿn"#Aÿs -  l 
 #ljAÿn:   Aj"# Aj-  AüË"j-   	lAÿn"$Aÿs #-  l  $ljAÿn:   Aj"# Aj-  AüË"j-   	lAÿn"$Aÿs #-  l  $ljAÿn:   E
  AM
	 AjAÿ:   Aj! Aj!  I
   "Aj"" G
  Aj" G
 A! (D! (@!@ - AqE
   )7X B 7P  (,"   AÐ j  A  ( (<      A ¦Å6  ("E
   Aj"6A ! 
  ( (    (l"E
  !@  (p"F
 @  AljÉ	"G
  (l!  6p  (t kâC Aðj$  ž}}}}}}}}}}# AÀ k"$ A!@ (A,l"
E
  ( " 
j!  r!@@@  ( (ï
"E
  A(jAj"
 ) 7  B 7,  84  8( Aj  A(j‹ 
 AjAj) 7  A(jAj" AjAj) 7   )7( *! *! *! *!  * " * "” *" *4"”’8  
   
* "”  *<"”’’8    ”  ”’84   *("”  *,"”’8,   ”  ”’8(    ”  ”’’8< Aj £	"
 A(j¯	@ E
   
/  "; Av!@ E
  A:   AÀ r:    
     AjžE
@ 	E
  	 
 ¦	 
¤	 A,j" G
   
¤	A ! AÀ j$  ”@@@  E
  (AM
 -  AüË"j-  l"Aÿn! @@ ( "Aj-  "
  /  ! Aj - :    ;   AÿI
  -    AÿlAÿÿq   j   lAÿÿqAÿnk"Aÿqn" lAÿ  k" -  ljAÿm:   Aj" -   l  -  ljAÿm:   Aj" -   l  -  ljAÿm:   !  Aj  :   -  AüË"j-  l"Aÿn!  AÿI
  (AM
 ( " -    lAÿ  k" -  ljAÿn:   Aj" -   l  -  ljAÿn:   Aj" -   l  -  ljAÿn:   ‡@@  
  (AM
 ( "Aÿ -  AüË"j-  lAÿn" k" -  l   -  ljAÿn:   Aj" -   l  -  ljAÿn:   Aj" -   l  -  ljAÿn:   -  AüË"j-  l"Aÿn! @ AÿI
  (AM
@@ ( "Aj-  "
  /  ! Aj - :    ;    -    AÿlAÿÿq   j   lAÿÿqAÿnk"Aÿqn" lAÿ  k" -  ljAÿm:   Aj" -   l  -  ljAÿm:   Aj" -   l  -  ljAÿm:   !  Aj  :   N# Ak"$  Aj¢	" ª	 ( ! A;     A  A  Ajž ¤	 Aj$ œ# Ak"$  Aj¢	" ( A¨	@ ( ( "kAM
 A!@   AtjA ¨	 Aj" ( ( "kAuI
  ( ! A;     A  A  Ajž ¤	 Aj$ e# A0k"$  Aj»	" 8 Aj¢	" ª	 ( ! A;      A   Ajž ¤	 ¼	 A0j$ n# A0k"$  A$j¢	" A¨	  A ¨	 Aj»	" 8 ( ! A;      A   Ajž ¼	 ¤	 A0j$ N# Ak"$   Œ! Aj¢	" ª	 A;     A  A  Ajž ¤	 Aj$ º}}}# AÀ k"$  *! *!  * 8  *8@ C   ?’"	 C   ¿’_E
   k²  “•! At!
@  	8  	8 *! A4j¢	" AjA¨	  AjA ¨	 Aj»	! A€€€þ6  A; @@  	 “”"‹C   O]E
  ¨!A€€€€x!     A  
  j"Atr Atr r Ajž ¼	 ¤	 	C  €?’"	 *C   ¿’_
  AÀ j$ õ	
}}}}}}}}}}}# AÐ k"	$ @ C    _
  C   ?”!
 *! *! *!
 * !@@@@   	A4j¢	"   
 «	   ’  ’ 
 “  “«	  Œ! 	A;     A  A  	Ajž ¤	 	A4j»	! 	AÝC"6( 	 Aj"60 B€€€‚„€€ À 7  	 6,  	A(j¾	@ 	(("E
  	 6,  	(0 kâC  8 	Aj¢	! 	 
 ’"8 	 
 ’"8  	AjA¨	 	  
“"8 	 8  	AjA ¨	 	 8 	 
 
“"
8  	AjA ¨	 	 8 	 
8  	AjA ¨	 	 8 	 8  	AjA ¨	  Œ! 	A;      A   	Ajž ¤	 ¼	 	A4j»	" 
8 	Aj¢	! 	 
 ’"8 	 
 ’"8  	AjA¨	 	  
“"8 	 8  	AjA ¨	 	 8 	 
 
“"
8  	AjA ¨	 	  “"8 	 
 “"8  	AjA ¨	 	 8 	  ’"8  	AjA ¨	 	  ’"8 	 8  	AjA ¨	 	 8 	 8  	AjA ¨	  Œ! 	A;       A  	Ajž 	Aj¢	! 	 8 	 
8  	AjA¨	 	 8 	 
8  	AjA ¨	 	 8 	 8  	AjA ¨	 	 8 	 8  	AjA ¨	 	 8 	 8  	AjA ¨	 	 8 	 8  	AjA ¨	 	 8 	 
8  	AjA ¨	  Œ! 	A;       A  	Ajž 	Aj¢	"   
 «	    
 «	  Œ! 	A;       A  	Ajž ¤	 ¤	 ¤	 ¼	 	A4j»	" 8 	Aj¢	! 	 
 ’"8 	 8  	AjA¨	 	 8 	 
8  	AjA ¨	  Œ! 	A;      A   	Ajž ¤	 ¼	 	AÐ j$    (,"    ( (T    (,"    ( (X     6  (," ( (    p# Ak"$ @  ( "(,"E
  A  ( (   (," ( (0  A$j Aj) 7   ) 7  A 6  Aj$   '   A 6   6   6  B 7  A :    X   ) 7   Aj Aj( 6   Aj Aj) 7    ("6@ E
   (Aj"6 
    J  (!  A 6@@ E
  ("E
  Aj"6 
   ( (     Ø# AÀ k"$ @@@  -  
   Aj õ A0jAj" Aj) 7   ) 70 A jAj"  Aj) 7    )7 @  ("E
   (Aj"6 E
 AjAj ) 7  Aj ) 7   )07  ) 7    Aj  À AÀ j$  ¿# Ak"$   A:     ) 7  Aj" Aj) 7   Aj õ@@@@@@ ( "  ("L
   ("  ("J
  A :  @  ( G
   (G
   (G
   (G
   (!   6 E
 (" E
   Aj" 6  E
A,ÝC¿" (Aj"6 E
  (!   6@ E
  ("E
  Aj"6 
   ( (    (  (  (k  (  (kAˆÀE
  ("  (N
   ( ( k!@ Aj  (   (kÆ (! (!	 Aj   (kÆ (" I
  ("
  ("k"  kK
  I
@ 
 F
  	 ( j ×6 Aj"  (H
  E
 (" E
   Aj" 6 !  
  ( (   Aj$  ¼~# Aà k"$  (! (!  6L  6H   j"6T   j"6P@@  L
   L
  /AˆG
@@  -  
  AjAj  Aj) 7   )! AjAj AÈ jAj) 7   7  )H7   Aj Aj À AÀ j"  Aj) 7    )78  Aj! A8j AÈ jõ@@@ (  (8L
  (D (<J
  A :    (!  A 6@ E
  ("E
  Aj"6 
   ( (    )87  Aj A8jAj) 7 A,ÝC¿"	 	(Aj"6 E
 	 (@ (8k (D (<kAˆÀE
@ (<"
 (DN
 @ A0j  ( 
  (kÆ A(j  
 kÆ AØ j 	 
 (<kÆ@ (8" (@N
  (\! (X!@  ( k" (4O
  k"
 (,O
  (8k" O
  j (( 
j-   (0 j-  lAÿn:   Aj" (@H
  
Aj"
 (DH
   )87  Aj A8jAj) 7   (!   	6@ E
  ("E
  Aj"6 
   ( (   E
 ("E
  Aj"6 
   ( (   Aà j$  ù ! ! !@@@@@@@@@@@@@   	
  j  lA~mj    H    JAÿ! AÿF
 AÿlAÿ km"Aÿ AÿH@ 
 A AÿAÿ kAÿl m"Aÿ AÿHk ! !@ Aÿ J
   lAtAÿm AtA~j" j  lA~mj At! @ Aÿ J
 Aÿ k lAÿ  klAÿƒ|m j AÿqAüÍ"j-   k  A~jlAÿm j  k" Au"s k  j  lAtA~mj   lAÿm! -   A :   A 6  B 7  AjB 7   AjA :       (!  A 6@ E
  J  Ö~# Ak"$    :    6   ;   ; A !@ AI
  AF
 @ E
  A€rAˆF
A!@@ Aj @ Aÿ}j  @@ AˆF
  AG
 AvAÿqA;l AÿqAlj Av"AÿqAljAä n A€þqr!A!@  - AF
   A:    ; Av!@  - AF
   A:    : @  - E
   A :    6 AˆF
   ) "7   7   Æ Aj$  ¢  B 7  (!  A 6@ E
  JA  -  "t!  /!@@@ ("E
 @ AÿÿqAG
    6  A6 AW!  (!   6@ E
  J  (! ( !  (!A ! @   F
   F
   j   Atj( "AvAÿqA;l AÿqAlj AvAÿqAljAä n:    Aj"  G
     6  A6 AW!  (!   6@ E
  J  (!  K
  ( I
 AK
  ( A t×6@ AÿÿqAG
    6  A6 AW!  (!   6@ E
  J  (!  (!@ AF
 A ! @   F
   j  :    Aj"  G
   E
 A :   AF
 AjAÿ:     6  A6 AW!  (!   6@ E
  J  (!  (!@ AF
 A ! @   F
   Atj  At  Atr  r6   Aj"  G
   E
 A€€€x6  AF
 AjA6  Ö~~~# Aà k"$ @@@  / Ahj	    ) "7X  ) "7P  ) "7H  7(  7   7   A(j A j  AjÈ  ) "7@  ) "78  ) "70  7  7  7    Aj Aj  É Aà j$ æ;~~~# A°k"$ @@  / "Ahj	   AL
  Av!@@@@@@  /"!  Aÿ}j  - AF
 E
 ( ! ( ! ( !	A !  ("
A|qAG! 
AF!@ Aj-  A;l -  Alj Aj-  AljAä n!  	-  !@@ 
     !  
E
  
   Â! @ E
   j-  "AÿF
  Aÿs 	-  l   AÿqljAÿn!  	  :    j! 	Aj!	 Aj" G
    - AF
 ) "§!@ ) "
BÿÿÿÿV
   B ˆ§K
 E
 Aÿ Ø6 E
 Aq! 
§!	@ AF
  Aþÿÿÿq!A ! @  	-  " -  "j  lAÿnk:   Aj" 	Aj-  " -  "j  lAÿnk:   	Aj!	 Aj!  Aj"  G
  E
  	-  "	 -  " j 	  lAÿnk:   Av! (!  (!	@  - AG
 @ 	
 @ E
  E
 ( ! ( !  ( !	A !@@  j-  "E
 @@ AÿG
  	Aj  -  :   	Aj  Aj-  :    Aj-  ! 	Aj"
  -   l Aÿs" 
-  ljAÿn:   	Aj"
  Aj-   l  
-  ljAÿn:    Aj-   l  	-  ljAÿn! 	 :     j!  	 j!	 Aj" G
   E
 Aq! ( !	 ( ! @ AF
  Aþÿÿÿq!A !@  Aj 	-  :    Aj 	Aj-  :     	Aj-  :     j" Aj 	 j"	-  :    Aj 	Aj-  :     	Aj-  :   	 j!	   j!  Aj" G
  E
  Aj 	-  :    Aj 	Aj-  :     	Aj-  :   ) ! ) !
@ E
  E
 ) §! §! 
§! A ! 	A|qAF!@@ -  "E
 @@ 
   Aj!  	 -  " -  Â l  Aÿs"
ljAÿm:    Aj!  	 -  " Aj-  Â l 
 ljAÿm:   	  -  " Aj-  Â!   -  ": £   Aj"-  : ¢   Aj"-  : ¡ 	  A¡j A¤jÊ  (¤ l Aÿs"
 -  ljAÿm:    (¨ l 
 -  ljAÿm:   (¬!    l 
 ljAÿm:    j! Aj!   j!  Aj" G
   E
 §! 
§! A ! 	A|qAF!@@@ 
   Aj!  	 -   -  Â:    Aj!  	 -   Aj-  Â:   	  -   Aj-  Â!   -  : £   Aj"-  : ¢   Aj"
-  : ¡ 	  A¡j A¤jÊ 
 (¤:    (¨:   (¬!   :    j!   j!  Aj" G
  @ 	
 @ E
  E
 ( ! ( !  ( !	A !@@  j-  "E
 @ AÿG
  	  /  ;   	Aj  Aj-  :   	  -   l Aÿs" 	-  ljAÿn:   	Aj"
  Aj-   l  
-  ljAÿn:   	Aj"
  Aj-   l  
-  ljAÿn:     j!  	 j!	 Aj" G
   ( !	 ( ! @  F
  E
 Aq!@ AI
  Aüÿÿÿq!A !@   	/  ;    Aj 	Aj-  :     j" Aj 	 j"	Aj-  :     	/  ;     j" Aj 	 j"	Aj-  :     	/  ;     j" Aj 	 j"	Aj-  :     	/  ;   	 j!	   j!  Aj" G
  E
A !@   	/  ;    Aj 	Aj-  :   	 j!	   j!  Aj" G
    l"E
   	 Ö6 ) ! ) !
@ E
  E
 ) §! §! 
§! A ! 	A|qAF!@@ -  "E
 @@ 
    	  -  " -  Â l  Aÿs"
ljAÿm:    Aj!  	 -  " Aj-  Â l 
 ljAÿm:   	  Aj-  " Aj-  Â! 	    A¤jÊ   (¤ l Aÿs"
  -  ljAÿm:    Aj" (¨ l 
 -  ljAÿm:    Aj-  ! (¬!  Aj  l 
 ljAÿm:    j!   j!  Aj! Aj" G
   E
 §! 
§! A ! 	A|qAF!@@@ 
    	  -   -  Â:    Aj!  	 -   Aj-  Â:   	  Aj-   Aj-  Â! 	    A¤jÊ   (¤:    Aj (¨:   (¬!  Aj :    j!   j!  Aj" G
   A G
  ) ! (!  (!@  - AG
 @ 
 @ E
   7˜  ) "
7  ) "7ˆ  7   
7  7 A j Aj   AjË  B"ˆ§K
  6„  >€  ) "7x  )€7  7  Aj  Ì@ E
  E
 ) §! ) §!	 §!A ! A|q"AG!@@@ Aj"
-  " 
  Aj 	-  :   Aj 	Aj-  :    	Aj-  :   -  "E
  
   j   lAÿnk":   Aÿl Aÿqn!@@ 
   -  : £  Aj-  : ¢  Aj-  : ¡  	 A¡j A¤jÊ 	-  ! (¤!
  Aj-   	-  "Â!
 Aj" 
  l   Aÿs"
ljAÿm lAÿ k" -  ljAÿm:   	Aj-  !@@ AF"
   Aj-   Â! (¨! Aj"   l 
 ljAÿm l  -  ljAÿm:   	Aj-  !@@ 
   -   Â! (¬!    l 
 ljAÿm l  -  ljAÿm:   	 j!	 Aj! Aj! Aj" G
    B"ˆ§K
 E
 §" Atj! ) "B ˆ§! §!	 A|qAG! Atj!@ - !  Aÿ: @@  
   	-  :   	- :   	- :    O
@@ 
  - ! - ! -  ! 	-  !
 	- ! 	- !@@@@   A !A !A !@   
  
K"  K" 
  
 I"   I"F
      K"  K    I"   Ik" 
 kl  k"m!   kl m!   kl m! A;l Alj AljAœm A;l Alj AljAä nj" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j!A !A !A !@     K"  K"    I"   I"F
   k   
  
K"  K 
  
 I"   Ik"l  k"m!  k l m!  k l m! A;l Alj AljAœm A;l Alj AljAä nj" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j! A;l Alj AljAä n A;l Alj 
AljAä nk" j"  j"  
j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j! A;l Alj 
AljAä n A;l Alj AljAä nk" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j!  -   	- Â!  -  	- Â!  -  	-  Â! 	-  !
    l 
  Aÿs"ljAÿm:    	- l   ljAÿm:    	- l   ljAÿm:    I
  k! 	 j!	 Aj" G
  @ 
 @ E
   7p  ) "
7h  ) "7`  7H  
7@  78 AÈ j AÀ j   A8jÍ  B"ˆ§K
  6\  >X  ) "7P  )X70  7( A0j A(j Î@ E
  E
 ) §! ) §!	 §!A ! A|q"AG!@@@ Aj"
-  " 
   	/  ;   Aj 	Aj-  :   -  "E
  
   j   lAÿnk":   Aÿl Aÿqn!@@ 
   	  A¤jÊ 	-  ! (¤!
  -   	-  "Â!
  
  l   Aÿs"
ljAÿm lAÿ k" -  ljAÿm:   	Aj-  !@@ AF"
   Aj-   Â! (¨! Aj"   l 
 ljAÿm l  -  ljAÿm:   	Aj-  !@@ 
   Aj-   Â! (¬! Aj"   l 
 ljAÿm l  -  ljAÿm:   	 j!	 Aj! Aj! Aj" G
    B"ˆ§K
 E
  §" Atj! ) "B ˆ§! §!	 A|qAG! Atj!@ - !  Aÿ: @@  
   	-  :    	- :   	- :   I
@@ 
  -  ! - ! - ! 	-  !
 	- ! 	- !@@@@   A !A !A !@   
  
K"  K" 
  
 I"   I"F
      K"  K    I"   Ik" 
 kl  k"m!   kl m!   kl m! A;l Alj AljAœm A;l Alj AljAä nj" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j!A !A !A !@     K"  K"    I"   I"F
   k   
  
K"  K 
  
 I"   Ik"l  k"m!  k l m!  k l m! A;l Alj AljAœm A;l Alj AljAä nj" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j! A;l Alj AljAä n A;l Alj 
AljAä nk" j"  j"  
j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j! A;l Alj 
AljAä n A;l Alj AljAä nk" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j!  -  	- Â!  -  	- Â!  -   	-  Â! 	-  !
    l 
  Aÿs"ljAÿm:     	- l   ljAÿm:    	- l   ljAÿm:   I
  k! 	 j!	 Aj" G
  A°j$  ƒ<~~~~# Ak"$ @  / A G
   ) "B"ˆ§K
 @@@@@@@  /"!  Aÿ}j  - AF
  (! ) !@ ) "	BÿÿÿÿV
  B ˆ ­T
 E
 §! §" Atj!
 A|qAG! AF!@@ - " E
  - A;l -  Alj - AljAä n! -  !@@ 
    ! E
    Â! -  !  Aÿq  l  Aÿs AÿqljAÿn:   Aj! Aj" 
G
   	B ˆ ­"
T
 B ˆ 
T
 E
 	§! §! §" Atj! A|qAG! AF!@@ -  -  l"AÿI
  - A;l -  Alj - AljAä n!  Aÿn!
 -  !@@ 
     !  E
     Â!  -  !   Aÿq 
l 
Aÿs AÿqljAÿn:   Aj! Aj! Aj" G
    - AF
 ) !@ ) "	BÿÿÿÿV
  B ˆ ­T
 E
 §" Atj! §!@ - !@@ -  " E
  E
   j   lAÿnk!  :   Aj! Aj" G
   	B ˆ ­"
T
 B ˆ 
T
 E
 	§! §! §" Atj!@ -  -  l"Aÿn! @@ -  "
E
  AÿI
 
  j   
lAÿÿqAÿnk!    :   Aj! Aj! Aj" G
    ("
A|q! ) "B ˆ§An! §! ) "§! §!@  - AG
 @ BÿÿÿÿV
 @ AG
   K
 E
  Atj!@ Aÿ  
Ï Aj! Aj" G
  @ 
E
   K
 E
  Atj!@ Aÿ  
Ð Aj! Aj" G
    K
 E
  Atj! @@ - "E
 @@ AÿG
   -  :   - :  - !  -   l Aÿs" - ljAÿn:   -  l  - ljAÿn:  -  l  -  ljAÿn!  :   Aj! Aj"  G
  @ AG
  B ˆ ­T
  K
 E
  Atj! @  -    
Ï Aj! Aj! Aj"  G
   B ˆ! ­!@ 
E
   T
  K
 E
  Atj! @  -    
Ð Aj! Aj! Aj"  G
    T
  K
 E
  Atj!@@ -  -  lAÿn" E
 @@  AÿG
   -  :   - :  - !   -    l  Aÿs"
 - ljAÿn:   -   l 
 - ljAÿn:  -   l 
 -  ljAÿn!    :   Aj! Aj! Aj" G
  @ BÿÿÿÿV
 @ AG
   K
 E
  Atj!@ Aÿ  
Ñ Aj! Aj" G
  @ 
E
   K
 E
  Atj!@ Aÿ  
Ò Aj! Aj" G
    K
 E
  Atj! @@ - "E
 @ AÿG
   -  :    - :   - :   -   l Aÿs" -  ljAÿn:    -  l  - ljAÿn:   -  l  - ljAÿn:  Aj! Aj"  G
  @ AG
  B ˆ ­T
  K
 E
  Atj! @  -    
Ñ Aj! Aj! Aj"  G
   B ˆ! ­!@ 
E
   T
  K
 E
  Atj! @  -    
Ò Aj! Aj! Aj"  G
    T
  K
 E
  Atj!@@ -  -  lAÿn" E
 @  AÿG
   -  :    - :   - :   -    l  Aÿs"
 -  ljAÿn:    -   l 
 - ljAÿn:   -   l 
 - ljAÿn:  Aj! Aj! Aj" G
    ("
A|q! ) "B"ˆ!	 §! ) "§! §!@  - AG
 @ BÿÿÿÿV
 @ AG
  	 ­T
 E
  Atj!@@ - "E
  Aj 
  Ó  Aÿs"  - l ( ljAÿm:     - l ( ljAÿm:     -  l ( ljAÿm:   Aj! Aj" G
   ­!@ 
E
  	 T
 E
  Atj!@ Aÿ  
Ô Aj! Aj" G
   	 T
 E
  Atj! @@ - "E
 @@ AÿG
   -  :   - :  - !  -   l Aÿs" - ljAÿn:   -  l  - ljAÿn:  -  l  -  ljAÿn!  :   Aj! Aj"  G
  @ AG
  B ˆ ­"T
 	 T
 E
  Atj!@@ -  -  l" AÿI
  Aj 
  Ó   Aÿn" Aÿs" - l (  ljAÿm:    - l (  ljAÿm:    -  l (  ljAÿm:   Aj! Aj! Aj" G
   B ˆ! ­!@ 
E
   T
 	 T
 E
  Atj! @  -    
Ô Aj! Aj! Aj"  G
    T
 	 T
 E
  Atj!@@ -  -  lAÿn" E
 @@  AÿG
   -  :   - :  - !   -    l  Aÿs"
 - ljAÿn:   -   l 
 - ljAÿn:  -   l 
 -  ljAÿn!    :   Aj! Aj! Aj" G
  @ BÿÿÿÿV
 @ AG
  	 ­T
 E
  Atj!@@ - "E
  Aj 
  Õ  Aÿs"  -  l ( ljAÿm:      - l ( ljAÿm:     - l ( ljAÿm:  Aj! Aj" G
   ­!@ 
E
  	 T
 E
  Atj!@ Aÿ  
Ö Aj! Aj" G
   	 T
 E
  Atj! @@ - "E
 @ AÿG
   -  :    - :   - :   -   l Aÿs" -  ljAÿn:    -  l  - ljAÿn:   -  l  - ljAÿn:  Aj! Aj"  G
  @ AG
  B ˆ ­"T
 	 T
 E
  Atj!@@ -  -  l" AÿI
  Aj 
  Õ   Aÿn" Aÿs" -  l (  ljAÿm:     - l (  ljAÿm:    - l (  ljAÿm:  Aj! Aj! Aj" G
   B ˆ! ­!@ 
E
   T
 	 T
 E
  Atj! @  -    
Ö Aj! Aj! Aj"  G
    T
 	 T
 E
  Atj!@@ -  -  lAÿn" E
 @  AÿG
   -  :    - :   - :   -    l  Aÿs"
 -  ljAÿn:    -   l 
 - ljAÿn:   -   l 
 - ljAÿn:  Aj! Aj! Aj" G
   A G
   ("
A|q! ) "B"ˆ!	 §! ) "§! §!@  - AG
 @ BÿÿÿÿV
 @ AG
  	 ­T
 E
  Atj!@ - ! @@ - "
   -  :   - :  - !   :   :    Aÿq"E
  Aj 
  Ó - ! - ! -  ! (! (! (!    j  lAÿnk" :   Aÿl  AÿqnAÿq" Aÿs" - l  Aÿs"l  ljAÿm  ljAÿm:    - l  l  ljAÿm  ljAÿm:    -  l  l  ljAÿm  ljAÿm:   Aj! Aj" G
   ­!@ 
E
  	 T
 E
  Atj! @ - !@@ - 
   -  :   - :  - !  :   :   Aÿq"E
     
× Aj! Aj"  G
   	 T
 E
  Atj!
@ - !@@@ - " 
   -  :   - :  - !  Aÿq"E
  -   Aÿl   j   lAÿnk"AÿqnAÿq" l  Aÿs" - ljAÿn:   -   l  - ljAÿn:  -   l  -  ljAÿn!   :    :   Aj! Aj" 
G
  @ AG
  B ˆ ­"T
 	 T
 E
  Atj!@ -  -  lAÿn!@@ - " 
   -  :   - :  - !   :    :   AÿqE
  Aj 
  Ó - ! - ! -  ! (! (! (!    j   lAÿÿqAÿnk":   AÿlAÿÿq AÿqnAÿq"Aÿs" - l   Aÿs"l   ljAÿm ljAÿm:    - l  l   ljAÿm ljAÿm:    -  l  l   ljAÿm ljAÿm:   Aj! Aj! Aj" G
   B ˆ! ­!@ 
E
   T
 	 T
 E
  Atj!@ -  -  lAÿn! @@ - 
   -  :   - :  - !   :   :    Aÿq" E
      
× Aj! Aj! Aj" G
    T
 	 T
 E
  Atj!
@ -  -  lAÿn! @@@ - "
   -  :   - :  - !  AÿqE
  -    AÿlAÿÿq   j   lAÿÿqAÿnk"AÿqnAÿq" l  Aÿs" - ljAÿn:   -   l  - ljAÿn:  -   l  -  ljAÿn! !    :   :   Aj! Aj! Aj" 
G
  @ BÿÿÿÿV
 @ AG
  	 ­T
 E
  Atj!@ - ! @@ - "
   -  :    - :  - !   :   :   Aÿq"E
  Aj 
  Õ - ! - ! -  ! (! (! (!    j  lAÿnk" :   Aÿl  AÿqnAÿq" Aÿs" -  l  Aÿs"l  ljAÿm  ljAÿm:     - l  l  ljAÿm  ljAÿm:    - l  l  ljAÿm  ljAÿm:  Aj! Aj" G
   ­!@ 
E
  	 T
 E
  Atj! @ - !@@ - 
   -  :    - :  - !  :   :  Aÿq"E
     
Ø Aj! Aj"  G
   	 T
 E
  Atj!
@ - !@@@ - " 
   -  :    - :   - :  Aÿq"E
  -   Aÿl   j   lAÿnk"AÿqnAÿq" l  Aÿs" -  ljAÿn:    -   l  - ljAÿn:   -   l  - ljAÿn:   :  Aj! Aj" 
G
  @ AG
  B ˆ ­"T
 	 T
 E
  Atj!@ -  -  lAÿn!@@ - " 
   -  :    - :  - !   :    :  AÿqE
  Aj 
  Õ - ! - ! -  ! (! (! (!    j   lAÿÿqAÿnk":   AÿlAÿÿq AÿqnAÿq"Aÿs" -  l   Aÿs"l   ljAÿm ljAÿm:     - l  l   ljAÿm ljAÿm:    - l  l   ljAÿm ljAÿm:  Aj! Aj! Aj" G
   B ˆ! ­!@ 
E
   T
 	 T
 E
  Atj!@ -  -  lAÿn! @@ - 
   -  :    - :  - !   :   :   Aÿq" E
      
Ø Aj! Aj! Aj" G
    T
 	 T
 E
   Atj!
@ -  -  lAÿn! @@@ - "
   -  :    - :   - :   AÿqE
  -    AÿlAÿÿq   j   lAÿÿqAÿnk"AÿqnAÿq" l  Aÿs" -  ljAÿn:    -   l  - ljAÿn:   -   l  - ljAÿn:  !    :  Aj! Aj! Aj" 
G
  Aj$  ª Aj-  ! Aj-  ! Aj-  ! Aj-  ! -  ! -  !	A !A !A !
@@@@@  Atj A !A !A ! @   	  	K"
  
K" 	  	 I"
  
 I"
F
      K"  K    I"   Ik" 	 
kl  
k"	m!    
kl 	m!   
kl 	m! A;l Alj  AljAœm A;l Alj AljAä nj" j"
  j"   j"  J" 
 J! A;l 
Alj AljAä m!@    H" 
  
H"AJ
   k l  k"m j!  k l m j! 
 k l m j!
 A€H
  kAÿ k"l  k"m j!  k l m j! 
 k l m j!
A !A !A ! @     K"
  
K"    I"
  
 I"
F
   
k   	  	K"  K 	  	 I"   Ik"l  
k"m!   
k l m!  
k l m! A;l Alj  AljAœm A;l Alj AljAä nj" j"
  j"   j"  J" 
 J! A;l 
Alj AljAä m!@    H" 
  
H"AJ
   k l  k"m j!  k l m j! 
 k l m j!
 A€H
  kAÿ k"l  k"m j!  k l m j! 
 k l m j!
 A;l Alj AljAä n A;l 	Alj AljAä nk" j"
  j"  	j"  J" 
 J! A;l 
Alj AljAä m!@    H" 
  
H"AJ
   k l  k"m j!  k l m j! 
 k l m j!
 A€H
  kAÿ k"l  k"m j!  k l m j! 
 k l m j!
 A;l 	Alj AljAä n A;l Alj AljAä nk" j"
  j"  j"  J" 
 J! A;l 
Alj AljAä m!@    H" 
  
H"AJ
   k l  k"m j!  k l m j! 
 k l m j!
 A€H
   kAÿ k"l  k"m j!  k l m j! 
 k l m j!
  6  Aj 
6  Aj 6 ±@ AH
  ( !  ( ! ( !A ! @@   j-  "E
 @ AÿG
  Aj -  :   Aj Aj-  :   Aj-  ! AjAÿ:    :   Aj" -  " j  lAÿnk":   Aj" -   Aÿl Aÿqn"lAÿ k" -  ljAÿm:   Aj" Aj-   l  -  ljAÿm:    Aj-   l  -  ljAÿm:    j! Aj!  Aj"  G
 ¨@@  ("AÿÿÿÿqE
   ( " Atj! (! @  E
  ( "-  :   AF
  Aj-  :   AM
 Aj-  ! Aÿ:   :     I
   j6    k!  Aj" G
  @ AH
  ( !  ( ! ( !A ! @@   j-  "E
 @ AÿG
   /  ;   Aj Aj-  :   AjAÿ:   Aj" -  " j  lAÿnk":    -   Aÿl Aÿqn"lAÿ k" -  ljAÿm:   Aj" Aj-   l  -  ljAÿm:   Aj" Aj-   l  -  ljAÿm:    j! Aj!  Aj"  G
 ¨@@  ("AÿÿÿÿqE
   ( " Atj! (! @  E
  ( "-  :    AF
  Aj-  :   AM
 Aj-  ! Aÿ:   :    I
   j6    k!  Aj" G
  â	@  -  l"AÿI
  Aÿn! - ! - ! -  !  -  !  - !	  - !
A ! A !A !@@@@@ Atj A ! A !A !@ 
 	  	 K" 
 K"  	  	I" 
  
I"F
      K"    K    I"     Ik"   kl  k"m!   	 kl m!   
 kl m!  A;l  Alj AljAœm A;l Alj AljAä nj"  j"  j"  j"    J"  J! A;l Alj  AljAä m!@      H"	  	 H"	AJ
    k l  	k"	m j!   k l 	m j!  k l 	m j! A€H
   kAÿ k"	l  k"m j!   k 	l m j!  k 	l m j!A ! A !A !@     K"  K"    I"   I"F
   k 
 	  	 K"  
  K  	  	I"  
   
Ik" l  k"m!  k  l m!  k  l m!  A;l  Alj AljAœm A;l Alj AljAä nj"  j"  j"  j"    J"  J! A;l Alj  AljAä m!@      H"	  	 H"	AJ
    k l  	k"	m j!   k l 	m j!  k l 	m j! A€H
   kAÿ k"	l  k"m j!   k 	l m j!  k 	l m j! A;l Alj AljAä n 	A;l 
Alj AljAä nk"  
j"   	j"   j"    J"  J! A;l Alj  AljAä m!@      H"	  	 H"	AJ
    k l  	k"	m j!   k l 	m j!  k l 	m j! A€H
   kAÿ k"	l  k"m j!   k 	l m j!  k 	l m j! 	A;l 
Alj AljAä n A;l Alj AljAä nk"  j"   j"   j"    J"  J! A;l Alj  AljAä m!@      H"	  	 H"	AJ
    k l  	k"	m j!   k l 	m j!  k l 	m j! A€H
    kAÿ k"	l  k"m j!   k 	l m j!  k 	l m j!   l Aÿs" ljAÿm:     l  ljAÿm:    l  ljAÿm:  Ÿ@  -  l"AÿI
   -    - Â!  -   - Â!  -   -  Â!  Aÿn" Aÿs" - l   ljAÿm:    - l   ljAÿm:    -  l   ljAÿm:  â	@  -  l"AÿI
  Aÿn! -  ! - ! - !  -  !  - !	  - !
A ! A !A !@@@@@ Atj A ! A !A !@ 
 	  	 K" 
 K"  	  	I" 
  
I"F
      K"    K    I"     Ik"   kl  k"m!   	 kl m!   
 kl m!  A;l  Alj AljAœm A;l Alj AljAä nj"  j"  j"  j"    J"  J! A;l Alj  AljAä m!@      H"	  	 H"	AJ
    k l  	k"	m j!   k l 	m j!  k l 	m j! A€H
   kAÿ k"	l  k"m j!   k 	l m j!  k 	l m j!A ! A !A !@     K"  K"    I"   I"F
   k 
 	  	 K"  
  K  	  	I"  
   
Ik" l  k"m!  k  l m!  k  l m!  A;l  Alj AljAœm A;l Alj AljAä nj"  j"  j"  j"    J"  J! A;l Alj  AljAä m!@      H"	  	 H"	AJ
    k l  	k"	m j!   k l 	m j!  k l 	m j! A€H
   kAÿ k"	l  k"m j!   k 	l m j!  k 	l m j! A;l Alj AljAä n 	A;l 
Alj AljAä nk"  
j"   	j"   j"    J"  J! A;l Alj  AljAä m!@      H"	  	 H"	AJ
    k l  	k"	m j!   k l 	m j!  k l 	m j! A€H
   kAÿ k"	l  k"m j!   k 	l m j!  k 	l m j! 	A;l 
Alj AljAä n A;l Alj AljAä nk"  j"   j"   j"    J"  J! A;l Alj  AljAä m!@      H"	  	 H"	AJ
    k l  	k"	m j!   k l 	m j!  k l 	m j! A€H
    kAÿ k"	l  k"m j!   k 	l m j!  k 	l m j!   l Aÿs" ljAÿm:    l  ljAÿm:     l  ljAÿm:  Ÿ@  -  l"AÿI
   -   - Â!  -   - Â!  -    -  Â!  Aÿn" Aÿs" -  l   ljAÿm:     - l   ljAÿm:    - l   ljAÿm:  - ! - ! -  ! -  ! - ! - !A !  A 6  B 7 @@@@@@ Atj A !A !A !	@     K"
  
K"    I"
  
 I"
F
      K"  K    I"   Ik"  
kl  
k"m!	   
kl m!   
kl m! A;l Alj 	AljAœm A;l Alj AljAä nj" j"  j"  	j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j!A !A !	@     K"
  
K"    I"
  
 I"
F
   
k     K"  K    I"   Ik"l  
k"m!	  
k l m!  
k l m! A;l Alj 	AljAœm A;l Alj AljAä nj" j"  j"  	j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j! A;l Alj AljAä n A;l Alj AljAä nk" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j! A;l Alj AljAä n A;l Alj AljAä nk" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
   kAÿ k"l  k"m j!  k l m j!  k l m j!   6   6   6 Ÿ@  -  l"AÿI
   -    - Â!  -   - Â!  -   -  Â!  Aÿn" Aÿs" - l   ljAÿm:    - l   ljAÿm:    -  l   ljAÿm:   -  ! - ! - ! -  ! - ! - !A !  A 6  B 7 @@@@@@ Atj A !A !A !	@     K"
  
K"    I"
  
 I"
F
      K"  K    I"   Ik"  
kl  
k"m!	   
kl m!   
kl m! A;l Alj 	AljAœm A;l Alj AljAä nj" j"  j"  	j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j!A !A !	@     K"
  
K"    I"
  
 I"
F
   
k     K"  K    I"   Ik"l  
k"m!	  
k l m!  
k l m! A;l Alj 	AljAœm A;l Alj AljAä nj" j"  j"  	j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j! A;l Alj AljAä n A;l Alj AljAä nk" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
  kAÿ k"l  k"m j!  k l m j!  k l m j! A;l Alj AljAä n A;l Alj AljAä nk" j"  j"  j"  J"  J! A;l Alj AljAä m!@    H"   H"AJ
   k l  k"m j!  k l m j!  k l m j! A€H
   kAÿ k"l  k"m j!  k l m j!  k l m j!   6   6   6 Ÿ@  -  l"AÿI
   -   - Â!  -   - Â!  -    -  Â!  Aÿn" Aÿs" -  l   ljAÿm:     - l   ljAÿm:    - l   ljAÿm: … - !  -    - Â!  -   - Â!  -   -  Â!  - !  - !	  -  !
 - !    j  lAÿnk":   Aÿl AÿqnAÿq"Aÿs" - l 
  Aÿs"l   ljAÿm ljAÿm:    - l  	l   ljAÿm ljAÿm:    -  l  l   ljAÿm ljAÿm:  … - !  -   - Â!  -   - Â!  -    -  Â!  - !  - !	  -  !
 - !    j  lAÿnk":   Aÿl AÿqnAÿq"Aÿs" -  l 
  Aÿs"l   ljAÿm ljAÿm:     - l  	l   ljAÿm ljAÿm:    - l  l   ljAÿm ljAÿm: Ê~~~# Aà k"$  ) !@@  / AG
   7X  ) "7P  ) "	7H  7  7  	7    Aj Aj   Ú  7@  ) "78  ) "	70  7(  7   	7   A(j A j   AjÛ Aà j$ ç~~@  / AG
 @@@@@@  /"!  Aÿ}j  - AF
@  ("E
   (AG
  ("E
 AF
 ( ! ( ! ( ! -  !	 Aj-  !@  ("

  AH
A ! @  	    j"Am"j-   At kAjvAq!@ E
    j-  "AÿF
  Aÿs -  l  AÿqljAÿn!  :   Aj!  Aj"  G
   AH
A ! 
A|qAG! 
AF!@  	   j" Am"j-   At  kAjvAq! @@ 
  
 -  !  
 -    AÿqÂ! @ E
   j-  "AÿF
  Aÿs -  l   AÿqljAÿn!    :   Aj! Aj" G
    - AF
 ) "
§!@ ) "BÿÿÿÿV
   
B ˆ§K
 E
 Aÿ Ø6 AH
 Aq! §! @ AF
  Aþÿÿÿq!A !@   -  " -  "j  lAÿnk:   Aj"  Aj-  " -  "j  lAÿnk:    Aj!  Aj! Aj" G
  E
   -  " -  " j   lAÿnk:    ("	A G  (AGq!@  - AG
  
 ( ! ( ! ( !Aÿ!
  ("! !Aÿ!Aÿ!@@   	( " Aÿq! 	Aj( "Aÿq!  AvAÿq!  AvAÿq! AvAÿq! AvAÿq!
 AH
 Av!A ! @      j"Am"j-   At kAjvAq"!   ! 
  !@@ E
    j"	-  "AÿF
  Aj" Aÿs -  l  ljAÿn:   Aj" 	-  "Aÿs -  l  ljAÿn:   	-  "Aÿs -  l  ljAÿn! Aj :   Aj :    :    j!  Aj"  G
   
  (" E
  AF
 AH
 ( ! ( ! ( ! 	( !
 	Aj( !AA AøÿqA F!A ! @  
    j"Am"j-   At kAjvAq"Av! Av!@@ E
    j"	-  "AÿF
   Aÿs -  l Aÿq ljAÿn:   Aj" 	-  "Aÿs -  l Aÿq ljAÿn:   Aj" 	-  "Aÿs -  l Aÿq ljAÿn:    :   Aj :   Aj :    j!  Aj"  G
   A G
   ("A G  (AGq!@  - AG
  
 ( ! ( ! ( !Aÿ!Aÿ!
Aÿ!  ("! !@@   ( " Aÿq! Aj( "Aÿq!  AvAÿq!  AvAÿq! AvAÿq!
 AvAÿq! AH
A ! @      j"Am"j-   At kAjvAq"! 
  !   !@@@ E
    j-  "	E
 	AÿG
  :   Aj :   Aj :   AjAÿ:   Aj" -  " 	j  	lAÿnk":   Aj"Aÿ 	Aÿl Aÿqn"	k" -  l  	ljAÿm:   Aj"  -  l  	ljAÿm:     -  l  	ljAÿm:   Aj!  Aj"  G
   
  (" E
  AF
 AH
  ( !	 ( ! ( ! ( ! Aj( !
A ! @ 
     j"Am"j-   At kAjvAq"Av! Av!@@@ 	E
  	  j-  "E
 AÿG
  :   AjAÿ:   Aj :   Aj :   Aj" -  " j  lAÿnk":   Aÿ Aÿl Aÿqn"k" -  l Aÿq ljAÿm:   Aj"  -  l Aÿq ljAÿm:   Aj"  -  l Aÿq ljAÿm:   Aj!  Aj"  G
  æ
~~@  / AG
 @@@@@@  /"!  Aÿ}j  - AF
@  ("E
   (AG
 ( ! ( ! ( !@  ("
  AH
A !	@  -  j-  ! @ E
   	j-  "AÿF
  Aÿs -  l   ljAÿn!    :   Aj! Aj! 	Aj"	 G
   AH
A !	 A|qAG!
 AF!@  -  j-  ! @@ 

  
 -  !   -    Â! @ E
   	j-  "AÿF
  Aÿs -  l   AÿqljAÿn!    :   Aj! Aj! 	Aj"	 G
    - AF
 ) "§!@ ) "
BÿÿÿÿV
   B ˆ§K
 E
 Aÿ Ø6 AH
 Aq! 
§!@ AF
  Aþÿÿÿq!	A !@  -  "  -  "j   lAÿnk:   Aj"  Aj-  "  -  " j   lAÿnk:   Aj! Aj! Aj" 	G
  E
  -  " -  "j  lAÿnk:    ("	A G  (AGq!@  - AG
  
 AH
 Av! ( !
 ( !  ( !A !@  -  !@@ 	E
  	 Atj( ! At Atr r! Av! Av!@@ 
E
  
 j"-  "AÿF
  Aj" Aÿs -  l Aÿq ljAÿn:   Aj" -  "Aÿs -  l Aÿq ljAÿn:   -  "Aÿs -  l Aÿq ljAÿn! Aj :   Aj :    :    Aj!   j! Aj" G
   
 AH
 ( !
AA AøÿqA F! ( ! ( !A !@ 	 -  Atj( " Av!  Av!@@ 
E
  
 j"-  "AÿF
   Aÿs -  l  Aÿq ljAÿn:   Aj"  -  "Aÿs  -  l Aÿq ljAÿn:   Aj"  -  "Aÿs  -  l Aÿq ljAÿn:     :   Aj :   Aj :   Aj!  j! Aj" G
   A G
   ("A G  (AGq!	@  - AG
  	
 AH
 ( ! ( !  ( !A !	@  -  !@@ 
  ! !  Atj( "Aÿq! AvAÿq! AvAÿq!@@@ E
   	j-  "
E
 
AÿG
  :   Aj :   Aj :   AjAÿ:   Aj" -  " 
j  
lAÿnk":   Aj"Aÿ 
Aÿl Aÿqn"
k" -  l  
ljAÿm:   Aj"  -  l  
ljAÿm:     -  l  
ljAÿm:    Aj!  Aj! 	Aj"	 G
   	
 AH
  ( ! ( ! ( !A !	@  -  Atj( " Av!  Av!@@@ E
   	j-  "
E
 
AÿG
   :   AjAÿ:   Aj :   Aj :   Aj" -  " 
j  
lAÿnk":   Aÿ 
Aÿl Aÿqn"
k" -  l  Aÿq 
ljAÿm:   Aj"    -  l Aÿq 
ljAÿm:   Aj"    -  l Aÿq 
ljAÿm:   Aj! Aj! 	Aj"	 G
  ¸~~~# A k"$ @@  / AˆG
 @@@@@@  /"!  Aÿ}j  - AF
  - AG
 AH
  /" Aÿq!  Av! ) "	B ˆ§! ) "
B ˆ§! 	§! ( ! 
§!
A ! @   F
   j-   l!@   O
   
  j-  lAÿn!@ AÿI
   AÿÿqAÿn"Aÿs -  l  ljAÿn:   Aj!  Aj"  G
    - AF
  - AG
 AH
  - ! ) "	B ˆ§! ) "
B ˆ§! 	§!
 ( ! 
§!A ! @   F
 
  j-   l!@   O
     j-  lAÿn! AÿÿqAÿn!@@ -  "E
  AÿI
  j Aÿq lAÿnk!  :   Aj!  Aj"  G
    - ! ) !	@  - AG
  Aÿq
 ) !
  (! ) !   (" 6 AH
 Av!  Aÿq!  Av!  AvAÿq!  AvAÿq! 
B ˆ§!
 	B ˆ§! 	§! 
§! §!A !  A|qAG!@   F
    j-  l!@   
O
     j-  lAÿn! AÿÿqAÿn!@ AÿI
 @ 
   -  ":   Aj"-  :   Aj"-  : 
  Aj A
j AjÊ  Aÿs" -  l ( ljAÿm:     -  l ( ljAÿm:    ( l  ljAÿm:   Aj"-  !@ E
    Â!  Aÿs" -  l  ljAÿm:    Aj"-   Â!   -  l  ljAÿm:    -   Â!   -  l  ljAÿm:    Aÿs" l  ljAÿn:     -  l  ljAÿn:   Aj"  -  l  ljAÿn:    j!  Aj"  G
   Aÿq
 ) !
  (! ) !   (" 6 AH
 Av!  Aÿq!  Av!  AvAÿq!  AvAÿq! 
B ˆ§!
 	B ˆ§! 	§! 
§! §!A !  A|qAG!@   F
    j-  l!@   
O
     j-  lAÿn! AÿÿqAÿn!@ AÿI
 @ 
   Aj  AjÊ  Aÿs" -  l ( ljAÿm:   Aj"  -  l ( ljAÿm:   Aj"  -  l ( ljAÿm:   -  !@ E
    Â!  Aÿs" -  l  ljAÿm:    Aj"-   Â!   -  l  ljAÿm:    Aj"-   Â!   -  l  ljAÿm:    Aÿs" l  ljAÿn:   Aj"  -  l  ljAÿn:   Aj"  -  l  ljAÿn:    j!  Aj"  G
   A G
   - ! ) !	@  - AG
  Aÿq
 ) !
  (! ) !   ("6 AH
 B"ˆ§! §! Av"Aÿq! Av"Aÿq! Aÿq! Av! 
B ˆ§!
 	B ˆ§! 	§! 
§!A !  A|qAG!@   F
    j-  l!@   
O
     j-  lAÿn! AÿÿqAÿn!   F
@@   Atj"- "
   :   :   :   :   AÿI
    j Aÿq lAÿnk":  AÿlAÿÿq Aÿqn!@ 
  Aj  Aj Ó  Aÿq"Aÿs" - l ( ljAÿm:    - l ( ljAÿm:    -  l ( ljAÿm:   - !@ E
    Â!  -  Â!  -   Â!  Aÿq"Aÿs" - l  l Aÿs" ljAÿm ljAÿm:    - l  l  ljAÿm ljAÿm:    -  l  l  ljAÿm ljAÿm:    Aÿq"Aÿs" l  ljAÿn:    - l  ljAÿn:    -  l  ljAÿn:    Aj"  G
   Aÿq
 ) !
  (! ) !   ("6 AH
  B"ˆ§! §! Av"Aÿq! Av"Aÿq! Aÿq! Av! 
B ˆ§!
 	B ˆ§! 	§! 
§!A !  A|qAG!@   F
    j-  l!@   
O
     j-  lAÿn! AÿÿqAÿn!   F
@@   Atj"- "
   :   :   :   :   AÿI
    j Aÿq lAÿnk":  AÿlAÿÿq Aÿqn!@ 
  Aj  Aj Õ  Aÿq"Aÿs" -  l ( ljAÿm:     - l ( ljAÿm:    - l ( ljAÿm:  -  !@ E
    Â!  -  Â!  -  Â!  Aÿq"Aÿs" -  l  l Aÿs" ljAÿm ljAÿm:     - l  l  ljAÿm ljAÿm:    - l  l  ljAÿm ljAÿm:   Aÿq"Aÿs" l  ljAÿn:     - l  ljAÿn:    - l  ljAÿn:   Aj"  G
  A j$  ¯ ®~# A k"$ @@@@  / AG
 @@@@@@@@  /"!  Aÿ}j





  - AF
  - AG
 AH
	 ( ! ( !  /" Aÿq!	  Av!
 ( ! A !@@   j"Am"j-   At kAjvAqE
  
!@ E
  
  j-  lAÿn! E
    Aÿs  -  l  	ljAÿn:    Aj!  Aj" G
 
   - AF
  - AG
 AH
 ( !  - !
 ( ! ( ! A !@@   j"Am"j-   At kAjvAqE
  
!@ E
   j-   
lAÿn!@  -  "E
  E
  j Aÿq lAÿnk!   :    Aj!  Aj" G
 	   - !
@  - AG
  
Aÿq
 ( ! ( ! ( !  (!   ("6 Av! Av!
 Av! Av!	@  r
  	AÿF
 AH
 
Aÿq! Aÿq! Aÿq!A !  A|qAG!@@    j"Am"
j-   
At kAjvAqE
  	!@ E
    j-   	lAÿn! E
 @ 
   -  ":   Aj"
-  :   Aj"-  : 
  Aj A
j AjÊ  Aÿs"
 -  l ( ljAÿm:   
 
 
-  l ( ljAÿm:    ( l 
 ljAÿm:   Aj"-  !
@ E
    
 Â l Aÿs"
 
ljAÿm:   Aj!
 
  
-  " Â l 
 ljAÿm:     -  "
 Â l 
 
ljAÿm:    Aÿs"
 
l  ljAÿn:    
 -  l  ljAÿn:   Aj"
 
 
-  l  ljAÿn:    j!  Aj"  G
 	  
Aÿq
 ( ! ( ! ( !  (!   ("6 Av! Av!
 Av! Av!	@  r
  	AÿF
 AH
 
Aÿq! Aÿq! Aÿq!A !  A|qAG!@@    j"Am"
j-   
At kAjvAqE
  	!@ E
    j-   	lAÿn! E
 @ 
   Aj  AjÊ  Aÿs"
 -  l ( ljAÿm:   Aj"
 
 
-  l ( ljAÿm:   Aj"
 
 
-  l ( ljAÿm:   -  !
@ E
   
 Â!
  Aÿs"
 -  l 
 ljAÿm:    Aj"
-   Â! 
 
 
-  l  ljAÿm:    Aj"
-   Â! 
 
 
-  l  ljAÿm:    Aÿs"
 
l  ljAÿn:   Aj"
 
 
-  l  ljAÿn:   Aj"
 
 
-  l  ljAÿn:    j!  Aj"  G
   A G
  - !@  - AG
  Aÿq
 ( ! ) ! ( !  (!   ("	6 	Av!
 	Av! 	Av! B"ˆ§!
 §!@  r
  AÿF
 AH
 	Aÿq! 
Aÿq! Aÿq!A !  A|qAG!@   
F
@    j"Am"j-   At kAjvAqE
    Atj! !@ E
    j-   lAÿn!@ - "
   
:   	:   :   :     j Aÿq lAÿnk":  AÿlAÿÿq Aÿqn!@ 
  Aj  Aj Ó  Aÿq"Aÿs" - l ( ljAÿm:    - l ( ljAÿm:    -  l ( ljAÿm:  @ E
   -   Â!  -  Â!  -  Â!  Aÿq"Aÿs" - l  l Aÿs" ljAÿm ljAÿm:    -  l  l  ljAÿm ljAÿm:     l  ljAÿm l  - ljAÿm:   Aÿq"Aÿs" - l  ljAÿn:    - l  ljAÿn:    -  l  ljAÿn:    Aj"  G
   Aÿq
  ( ! ) ! ( !  (!   ("	6 	Av!
 	Av! 	Av! B"ˆ§!
 §!@  r
  AÿF
 AH
 	Aÿq! 
Aÿq! Aÿq!A !  A|qAG!@   
F
@    j"Am"j-   At kAjvAqE
    Atj! !@ E
    j-   lAÿn!@ - "
   :   :   
:   	:     j Aÿq lAÿnk":  AÿlAÿÿq Aÿqn!@ 
  Aj  Aj Õ  Aÿq"Aÿs" -  l ( ljAÿm:     - l ( ljAÿm:    - l ( ljAÿm: @ E
   -  Â!  -  Â!  -   Â!  Aÿq"Aÿs" - l  l Aÿs" ljAÿm ljAÿm:    - l  l  ljAÿm ljAÿm:    l  ljAÿm l  -  ljAÿm:    Aÿq"Aÿs" -  l  ljAÿn:     - l  ljAÿn:    - l  ljAÿn:   Aj"  G
  ¯  AH
A !@@   j" Am"j-   At  kAjvAqE
   
O
  Atj" Aÿ:    :    
:    	:   Aj" G
   AH
A !@@   j" Am"j-   At  kAjvAqE
   
O
  Atj"  
:    	:   Aÿ:    :   Aj" G
    AH
A ! @@    j"Am"
j-   
At kAjvAqE
   :   Aj :   Aj 
:    j!  Aj"  G
   AH
 A ! @@    j"Am"
j-   
At kAjvAqE
   
:   Aj :   Aj :    j!  Aj"  G
  A j$ b   A 6(  B 7  A„Ð"6   A,jÃ  A÷ jA :    Aï jB 7    Aç jB 7    Aß jB 7    A× jB 7    B 7 O  Ø@  (l"E
    6p J@  (`"E
    6d J@  (T"E
    6X J  A,jÄ  ((!  A 6(@@ E
  ("E
  Aj"6 
   ( (    (!  B 7@ E
  ("E
  Aj"6 
   ( (     
   ßAø âC‘@@  (" ( "F
 @ E
   (Aj"6 E
  (!   6 E
  ("E
  Aj"6 
   ( (     6   ( 6   (6   ( ( k6 (! (!   6$   8   ((!  A 6(    k6@ E
  ("E
  Aj"6 
   ( (  @ E
  -  AG
 @ ("E
   (Aj"6 E
  ((!   6( E
  ("E
  Aj"6 
   ( (     
6P   	: O   : N   : M   : L ‚# Ak"$    ;  (/! (!  ( "6   kAu6  - O!  (P!  ($!  )7 @  A,j      Å"E
 @  - LAG
 @@   ("/AvAqlAj"  (X  (T"k"M
   AÔ j  kÜ  (!  O
     j6X@ ("  (d  (`"k"M
   Aà j  kÜ  O
     j6d  * C  €?[
 @  (AA  - Lj( "  (p  (l"k"M
   Aì j  kÜ  O
     j6p Aj$  ™
}~~~# Ak"$ @@  * "C  €?[
 @@ ("E
  AH
 ( !A !	@ 	 F
  (l 	j!
@@  *   	j-  ³”"C  €O] C    `qE
  ©!A ! 
 :   	Aj"	 G
     (p  (l"	kK
 C  C”•!
 E
  	 
 Ø6  (p!	   (l"
6   	 
k6  A,j!	 ) !@@@  /" AF
   AˆG
  7ˆ  ) "
7€  ) "7x  7(  
7   7 	 A(j A j  AjÜ  7p  ) "
7h  ) "7`  7@  
78  70 	 AÀ j A8jA   A0jÙ  7X  ) "
7P  ) "7H  7  
7  7  	 Aj Aj  Ç Aj$  þ~# AÐ k"$ @@@  - LAG
   ) "7  7@    AjåA !A !A !@  (("E
  AÈ j   ( j  ((kÆ (L"  (  ((k"I
 (H j!  k! AÈ j  (  ( jÆ (H!@ (L"E
   ("A H
  (1  ­~"BÿÿÿÿV
  Bˆ§"I
  j!  k!  6<  68  ) "70  (!  6,  6(  )87   7  )(7   A j Aj  Ajã AÐ j$  ‘	~~# AÀ k"$   ("( ! /!  (!  - M!  (!	 A8j Å  	 Asj  j!
 AvAq! (8!	@@@ (<"E
  
A H
 ­ 
­~"B ˆ§
  ("A H
 ­ ¬~"
B€€€€Z
  
§"I
  k" §"I
 	 j j!	  - NAG
 A H
  (Aj¬ ­~"
B€€€€Z
  k 
§"I
A  k! 	 j!	A  k   - NAq!@  ("AH
  Aq! Aq!  (T!A ! AI! 	!@@ E
 A !A !A !@ 
 @   j"-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj! Aj" G
 @ E
 @   j-  :   Aj! Aj! Aj" G
   (!  j! Aj" H
 @@  (("
 A !A ! ( !  (d!  (`! A8j   (  ((kÆ 
  ((k" (<K
 (8 j!@@  - N
   (! A H
  ("Aj¬ ­~"
B€€€€Z
A  k!  
§j!  k! AH
 A !@  F
  j -  :    j! Aj"  ("H
   (X!   (T"60   k64 ) !
  6$  6   
7(  )07  
7  ) 7   Aj Aj  Ajã@  ("AH
  Aq! Aq!  (T!A ! AI!@@ E
 A !A !A !@ 
 @ 	 j" -  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj! Aj! Aj" G
 @ E
 @ 	 j -  :   Aj! Aj! Aj" G
   (! 	 j!	 Aj" H
  AÀ j$  ¸
|||||||| -  !A !	  A 6  B 7     (6@ 
 A@@  H
    6 @@ · ·£"
™"›"D      ðAc D        fqE
  «!
A !
A !	A  
 
 
Aj"K"AýÿÿÿO
   AtAj"6  k"A€€€€ nK
  Au q!    l"	6@ 	E
   Aj 	Ü ·!@ D      ð?c
 A!	 - Aq
   L
 Aj! 
AÿÿÿÿI!@@@ 
 ·" 
¢  " "   cœ"™D      àAcE
  ª!	A€€€€x!	  	  	J!	@@    cœ"™D      àAcE
  ª!A€€€€x!  (   ( k  (lj!@@    H" 	N
  E
   	  	H"	6  	6   	k N
  6  	6 A€€!
@@  	L
  Aj! Aj·!A€€!
D        !@   	· ¡ 
£" 	Aj"· ¡ 
£"  d""  d    "  c¡"D         D        d" D      ð@¢–! 	 ( "H
 	 (J
  	 kAtj 6  
 k!
  ¸D      ð¾¢ ! !	  G
  
AjA€€O
  ( "	H
  (J
   	kAtjAj 
6  ("	 ( "L
  	Aj"	6  	 kAtjAj"	 	(  
j6 A!	 Aj" G
  @  J
 A Aj! 
D      à?¢! Aq!
@ · 
¢    !  (   ( k  (lj!	@@ 
E
 @@ D      à¿ œ"™D      àAcE
  ª!A€€€€x!    J!@@ D      à? œ"™D      àAcE
  ª!A€€€€x!    H" k N
 	 6 	 6 @  J
  	A€€6 	Aj  ·¡D      à¿ D      ð@¢–"6  	A€€ k6@@ œ"™D      àAcE
  ª!A€€€€x!    H"    J"k N
 	A€€6 	 6 	 6 A!	 Aj" G
  	 Ó}|||# A k"$    ;    Aÿq6   ( - 6   ( /A	vAq:    ( "	6 	 	(Aj"
6@ 
E
  ( "	(!
   	("	6   
 	kAu6   ( (6 ( (!   6,   6(   6$   6    ) 70  A8j Aj") 7   B 7@  AÈ jB 7   AÐ jB 7   AØ jB 7   Aà jA 6   Aì j™	!	  B 7t  A : q  Aü jB 7   A„jB 7   AŒjA 6  Aj  ( (  ( kÈ@ - E
 @@ ("  (D  (@"k"
M
   AÀ j  
kÜ  
O
     j6D@ A G
   (D  (@"k"AH
  Aÿ Ø6    (  (8  (0kÆ6d  A  (8  (0kÆ6h@@ - AG
   A: n@ E
  -  Aq
   4  4 ~  Au"s k¬  Au"s kAm¬W
  	A:   	 (  6 @@D          (("²"» A J" ( ·   ("² •»"
¢"  (·  
¢"  d"›"
™D      àAcE
  
ª!	A€€€€x!	 (! (!   	6\@@   œ"™D      àAcE
  ª!A€€€€x!   6T@@D          (,"²"» A J" ·   ( "² •»"
¢"  ·  
¢"  d"›"
™D      àAcE
  
ª!	A€€€€x!	  AÔ j!   	6`@@   œ"™D      àAcE
  ª!A€€€€x!   6X  6  6 B 7  Ajõ@@@@  (Aj   (AG!AA  (AF!AA  - !   : p A j$    @  („"E
    6ˆ J  (L!  A 6L@ E
  J@  (@"E
    6D J  A 6$  (!  A 6@@ E
  ("E
  Aj"6 
   ( (     ;A !@  - qAG
 @   ê"
  A: q  ë  - qAF
  ó
# Ak"$ A !@  ((E
 @  ("  (t  ( ( 
   (Am!  (t"  (`N
A
!@@ A
 !@ 
  E
   ( ( 
  (t!A
! Aj  ("  ( (   (P"  (t  (Xk  (d"l"I
   kK
 (!  (L j!@@@@@@  - p  A !	  (0"
  (8N
@A !@  („ 
  (xk  (|lj"( " ("
J
  Aj!A ! !@  H
	A   Am"j-   At kAjvAqk   kAtj( Aÿlq j! Aj" 
L
  Av! 	 F
  	j :   	Aj!	 
Aj"
  (8H
  A !  (0"	  (8N
@A !@  („ 	  (xk  (|lj"( " ("J
  Aj!
A ! !@  H
 
  kAtj(   j-  l j! Aj" L
  Av!  F
  j :   Aj! 	Aj"	  (8H
  A !  (0"  (8N
@A !	A !A !@  („   (xk  (|lj"( " ("J
  Aj!  (!  (!A !
  / AÿÿqAG! !A !A !	@  H
   j-  "M
   kAtj( !  Atj( !@@ 
  Aÿq!
 Av! Av! Av! Av! Av!
 
 l 
j!
 Aÿq l 	j!	 Aÿq l j! Aj" L
  	Av!	 Av! 
Av!  O
  j :   Aj" O
  j :   Aj" O
  j 	:   Aj! Aj"  (8H
  A !  (0"  (8N
@A !
A !	A !A !@  („   (xk  (|lj"( " ("J
  Aj!A !
 !A !A !	A !
@  H
   kAtj(    lj"Aj-  lAÿn" 
j!
  Aj-  l 	j!	  Aj-  l j!  -  l 
j!
 Aj" L
  	Av!	 Av! 
Av! 
AÿlAv!
  O
  j :   Aj" O
  j :   Aj" O
  j 	:   Aj" O
  j 
:    j! Aj"  (8H
  A !  (0"  (8N
 @A !	A !A !@  („   (xk  (|lj"( " ("
J
  Aj!A !
 !A !A !	@  H
   kAtj( "   lj"Aj-  l 	j!	  Aj-  l j!  -  l 
j!
 Aj" 
L
  	Av!	 Av! 
Av!  O
  j :   Aj" O
  j :   Aj" O
  j 	:    j! Aj"  (8H
     (tAj"6t Aj!   (`H
 A ! A! Aj$  “
# A0k"$ @@  (,"E
  A(jB 7  AjAjB 7  B 7@ Aj   (4  (<  (   (X  (`  Aì jæE
   (Am!  (4"  (<N
 @ ($  (k (lj!  (@!@@@@  - p     (0"  (8N
 Aj!@  (P"	   (0k l"
I
A !@ ( " ("J
  	 
k!
  (L 
j!  (d!  (X!A !
 !@  H
	  k l"	 
O
	   kAtj(   	j-  l 
j!
 Aj" L
  
Av!  :    j! Aj"  (8H
    (0"  (8N
 Aj!@  (P"
   (0k l"I
A !A !A !
@ ( "	 ("J
  
 k!  (L j!  (d!  (X!A ! 	!A !
A !@  	H
   k l"I
  kAM
   	kAtj( "
  j"Aj-  l j! 
 Aj-  l 
j!
 
 -  l j! Aj" L
  Av! 
Av! Av!
  
:   Aj :   Aj :    j! Aj"  (8H
    (0"  (8N
  Aj!@  (P"   (0k l"I
@@ ( " ("L
 A !  k!  (L j!  (d!  (X!A !
 !A !A !A !	@  H
   k l"
I
  
kAM
   kAtj( "  
j"
Aj-  l 	j!	  
Aj-  l j!  
Aj-  l j!  
-  l 
j!
 Aj" L
 @ 	
 A ! Aj Aÿl 	n"Aÿ AÿH"A  A J:   Aj Aÿl 	n"Aÿ AÿH"A  A J:    
Aÿl 	n"Aÿ AÿH"A  A J:   	Av! Aj :    j! Aj"  (8H
   (4!  ($!  (D!
   (@"	6  
 	k6 ( (!
  )7   k Aj 
  Aj"  (<H
  ($"E
   6( J A0j$  ô~A !@  ((E
   (d"E
   (@  (DF
 A !  (`  (Xk"A H
  ­ ¬~"BÿÿÿÿV
  B€€€€ B€€€€T§"E
  AH!  (L!   6LA ! A  !@ E
  J   6P E
   Aø j  ((  (0  (8  (  (T  (\  Aì jæE
 A!  A: q    (X6t ¥   A 6   6   6  (  !   6   6   6  A j Aj) 7    ) 7@@@@ /"Aj  AG
  Aˆ;(    A;(   ( (F
 A!   ;(  r  (!  A 6@ E
  èAâC  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    A 6    ±	# A k"$ A !@@  (E
   (E
 @@  ("/AG
  ( (F
   /(!  ( !  (!  ( !  (!  ($! Aj A èš	 Aj  (Aèš	 - AÿG
 - AÿG
  k!  k! A€AV"6  A€j"	6A ! A A€Ø6!  	6@  Atj -  - "k lAÿm jAt -  - "k lAÿm jAtr -  - "k lAÿm jrA€€€xr6  Aj"A€G
     Aÿÿq Aj ( ( !@ ("E
   6 JA ! 
  /(!  ( !  (!  ( !  (!  ($!A ! A 6 B 7    k  k   ( ( !@ ( "E
   6 J E
  ð! A j$   ŸAÝC  (   /(  (  (  Aj  Aj"  Ajç!  (!   6@ E
  èAâC  (! ì@@ ( "("E
 A! (AÀ„= mN
A !  (A é  @  (" 
 A    é'@  (" E
     (Aj"6 
       A 6  AœÐ"6   J  (!  A 6@@ E
  ("E
  Aj"6 
   ( (     O  (!  A 6@@ E
  ("E
  Aj"6 
   ( (    AâC   (!  A 6 T ( ! A 6   (!   6@@ E
  (" E
   Aj" 6  
   ( (   V# Ak"$  Aj  ( Æ@@ ("E
   (" I
  E
  ( (   ×6 Aj$  # Ak"$ A,ÝC¿" (Aj"6@ E
 @@    À"E
 @ ( " ("F
   6  6  (6 A 6 B 7   Ajì ("E
   6 J  (!   6 E
 ("E
  Aj"6 
  ( (   ("E
  Aj"6 
   ( (   Aj$   ’}}}}}}}}}}# A€k"$    6    ) 7  Aj Aj) 7   Aj Aj) 7   B 7  A$jB 7   A,j"B 7   A4jB 7   A 6T  B 7L  B€€€€€€€À?7D  B€€€ü7<  AØ jó! (  !  A 6d   6` A8j  Aj”	 Aà j A8jþ AÐ jAj Aà jAj) 7   )`7P@ E
  AÐ j õ@@ (X" (P"L
  (\" (T"L
   Aà j!  )P7  Aj AÐ jAj) 7 @  *"	‹"
  *"‹"C   A•]E
   *‹"
C   ?]E
  
C   ?]E
  
  *"
‹C   A•]E
  AÐ jAj"  (`"k6    k6P   (d"k6\   k6T A8j AÐ j (h k" (l k" 
C    ^ C    ]ö  A8jAj) 7   )87PA,ÝC!@  ( "E
   (Aj"6 E
      AÐ j í!  (T!   6T@ E
  îA,âC  (T! ï  A6d@ CÍÌL=]E
   *‹CÍÌL=]E
   *!   (`"k6X   k6P   (d"k6\   k6T@@ 	 	Ž 	C    ^"	‹C   O]E
  	¨!A€€€€x!@@  Ž C    ^Œ"	‹C   O]E
  	¨!A€€€€x!A,ÝC!@  ( "E
   (Aj"6 E
      AÐ j í!  (T!   6T@ E
  îA,âC  (T! ï  A6d@@ 	 ‘7"‹C   O]E
  ¨!A€€€€x! E
 @@  *"
  *"‘7"
‹C   O]E
  
¨!A€€€€x! E
   *!
  *!   ²"•"C    ”"  ²"•"“8D  	 •"C    ”" 
 •"	“8@   C    ”’8<   	C    ”’88     ”’’8L  
  	 ”’’8H A j A8j‹	 Að j A j  AÐ j÷•	 Aj Að jý AjóE
   6|  6x B 7p Aj Að jõ AjóE
   A<j" ) 7  Aj A jAj) 7  Aj A jAj) 7   Aj"Aj AjAj) 7   )7 A,ÝC!@  ( "E
   (Aj"6 E
       í!  (T!   6T@ E
  îA,âC  (T! ï  A6d A€j$    t  AØ jô  (T!  A 6T@ E
  îA,âC  ( !  A 6 @@ E
  ("E
  Aj"6 
   ( (     VA !@  (dE
 A!  (T ñ
 A !@@@@  (d       ý    þA ! ý# Ak"$ @@  (\"E
 @@ (Aj   ( (  @  (\"E
   (Aj"6 E
    *C    ^  *C    ]ò6  AØ j Aj÷ (!  A 6@  E
   ("E
   Aj"6 
     ( (   (" E
   Aj" 6  
   ( (   Aj$  «}}}}}}}# AÀ k"$ @@  (\"E
 @@ (Aj   ( (  A,ÝC¿" (Aj"6 E
  6<  (Tò"("E
AˆA  - Aq!  Aj"6@ 
   ( (  @@   (4  (,k  (8  (0k À
  A 6<  *L!  *P!  (,!  (0!  *D!	  *<!
   *@"C    ”  *H"’80  	 
C    ”’8,   C    ”’8(  
 	C    ”’8$    ²"
”  ²"”’’88   
 
” 	 ”’’84 A$jA   (k²A   ( k²Ž	  6  A$j6@  (\"E
   (Aj"6 E
 Aj Å  (6@@  (\"E
  (Aj"E
  ( 6 A (!  ( 6  E
  Aj"6@ 
   ( (   ("E
  Aj"6@ 
   ( (  @@  (\"E
  (Aj"
A ("E
  Aj"6 /A€q!@ 
   ( (  @@ E
    Ajÿ@@  (\"E
  (Aj"E
A ("E
  Aj"6 /!@ 
   ( (  @ AøqAG
    Aj€   Aj  AvAq  AØ j A<j÷ (<! A 6< E
 (" E
   Aj" 6  
   ( (   AÀ j$  ì}}}}}}}}}}}# Ak"$  ("* C  €C”•! *C  €C”•! *C  €C”•! *C  €C”•! *C  €C”•! *C  €C”•!@  (8  (0kAH
  ²!	 ²!
 ²! ²! ²!
 ²!A !@ Aj (  Æ@  (4"  (,"kAH
  (! 
 ³"”! 
 ”!A !@@@  ³"” ’ 	’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!@@  ” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@ AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
  AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
    ($  (k"J
    ((  ( k"J
   A€j  A€~K" (" ("   Fk" Aj"  Flj"   Fk" Aj"  F"j-  lAÿ k"  j-  ljAvAÿq A€j  A€~K"l    lj" j-  l   j-  ljAvAÿqAÿ kljAv:    (,!  (4! Aj! Aj"  kH
  Aj"  (8  (0kH
  Aj$ •
}}}}}}}}}}}# Ak"$ @@@  (\"E
  (Aj"E
 ( (F!A ("E
 ( (F!  Aj"6@ 
   ( (  @@ E
 A !@ Aj Atj A‚lA€€€xj6  Aj Ar"Atj A‚lA€€€xj6  Aj Ar"Atj A‚lA€€€xj6  Aj Ar"Atj A‚lA€€€xj6  Aj"A€G
  @@  (\"E
  (Aj"E
 ( ("k!A ("E
 ( ("k!  Aj"6@ 
   ( (   AÿM
 Aj A€Ö6 ( /! ("* C  €C”•! *C  €C”•! *C  €C”•! *C  €C”•! *C  €C”•!	 *C  €C”•!@  (8  (0kAH
  AvAq!
 ²! ²! ²!
 	²! ²! ²!A !@ Aˆj (  Æ@  (4"  (,"kAH
  (ˆ!  ³"”!  ”!A !@@@ 
 ³"” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!@@  ” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@ AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
  AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
    ($  (k"	J
    ((  ( k"J
   Aj A€j  A€~K" (" ("   Fk" Aj"  Flj"   	Fk" Aj"  	F"	j-  lAÿ k"  j-  ljAvAÿq A€j  A€~K"l    lj" 	j-  l   j-  ljAvAÿqAÿ kljAvAüqj( 6   (,!  (4!  
j! Aj"  kH
  Aj"  (8  (0kH
  Aj$  ¸}}}}}}}}}}}# Ak"$  ( /!@@@  (\"E
  (Aj"
A ("E
 Av!  Aj"6 /A€q!@ 
   ( (   Aq!	@@ 
  ("* C  €C”•! *C  €C”•! *C  €C”•! *C  €C”•! *C  €C”•!
 *C  €C”•!  (8  (0kAH
 ²! ²! ²!
 
²! ²! ²!A !@ Aj (  Æ@  (4"  (,"kAH
  (!  ³"”!  ”!A !@@@ 
 ³"” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!@@  ” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@ AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
  AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
    ($  (k"J
    ((  ( k"
J
   A€j  A€~K" (" ("   
Fk" Aj"  
Flj"
   Fk" Aj"  F l"j-  lAÿ k" 
  l"j-  ljAvAÿq A€j  A€~K"l    lj" j-  l   j-  ljAvAÿqAÿ k"ljAvAÿq  
Aj" j-  l   j-  ljAvAÿq l  Aj" j-  l   j-  ljAvAÿq ljA€þqr  
Aj"
 j-  l  
 j-  ljAvAÿq l  Aj" j-  l   j-  ljAvAÿq ljAtA€€üqrA€€€xr6   (,!  (4!  	j! Aj"  kH
  Aj"  (8  (0kH
   ("* C  €C”•! *C  €C”•! *C  €C”•! *C  €C”•!
 *C  €C”•! *C  €C”•!  (8  (0k!@ A G
  AH
 ²! 
²! ²!
 ²! ²! ²!A !@ Aj (  Æ@  (4"  (,"kAH
  (!  ³"”!  ”!A !@@@ 
 ³"” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!
@@  ” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@ AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
  AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
    ($  (k"J
    ((  ( k"J
   A€j  A€~K" (" ("   Fk" Aj"  Flj"Aj"   Fk" Aj"  F l"j-  lAÿ k"   l"j-  ljAvAÿq 
A€j 
 
A€~K"
l    lj"Aj" j-  l   j-  ljAvAÿqAÿ 
k"ljA€þq   j-  l   j-  ljAvAÿq 
l   j-  l   j-  ljAvAÿq ljAvAÿqr  Aj" j-  l   j-  ljAvAÿq 
l  Aj" j-  l   j-  ljAvAÿq ljAtA€€üqr  Aj" j-  l   j-  ljAvAÿq 
l  Aj" j-  l   j-  ljAvAÿq ljAtA€€€xqr6   (,!  (4!  	j! Aj"  kH
  Aj"  (8  (0kH
   AH
  ²! 
²! ²!
 ²! ²! ²!A !@ Aj (  Æ@  (4"  (,"kAH
  (!  ³"”!  ”!A !@@@ 
 ³"” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!
@@  ” ’ ’C   C’"‹C   O]E
  ¨!A€€€€x! A€o!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@@ C  €;”"C   Ï C   Ï^"‹C   O]E
  ¨!A€€€€x!@ AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
  AÿÿÿÿA  C   Ï` CÿÿÿN_"A H
    ($  (k"J
    ((  ( k"J
   A€j  A€~K" (" ("   Fk" Aj"  Flj"Aj"   Fk" Aj"  F l"j-  lAÿ k"   l"j-  ljAvAÿq 
A€j 
 
A€~K"
l    lj"Aj" j-  l   j-  ljAvAÿqAÿ 
k"ljA€þq   j-  l   j-  ljAvAÿq 
l   j-  l   j-  ljAvAÿq ljAvAÿqr  Aj" j-  l   j-  ljAvAÿq 
l  Aj" j-  l   j-  ljAvAÿq ljAtA€€üqr  Aj" j-  l   j-  ljAvAÿq 
l  Aj" j-  l   j-  ljAvAÿq ljAtA€€€xqr6   (,!  (4!  	j! Aj"  kH
  Aj"  (8  (0kH
  Aj$     AØ jö¶	}}}}~~# A0k"	$    ( "
6 @@ 
E
  
 
(Aj"6 E
   6   ) 7  Aj Aj) 7   Aj Aj) 7   B 7   A(jÞ!  A¨jB 7   B 7    : ¹A !
  A : ¸   6´   8° 	Aj  Aj"
”	  A j! 	A j 	Ajý@@ E
  Aj! Aj! (! (!
 ( "Aj! Aj!A ! ( !   ( 6¬   6¨   6¤   
6   	A jõ@@  (¨  ( L
   (¬  (¤L
   *!@@  *"‹"C   ?`
  C    [
   *‹C   ?`
   *"C    \
@ ‹" C   A•]E
   *‹"C   ?]E
  C   ?]E
    *"‹C   A•]E
  	((! 	(,!
 	($!
 	( ! 	AjAj" Aj) "7  ) !  § k6  	 7 	 § k6 	 	( 
k6 	 	( 
k6 	 	Aj  k" 
 
k"
 C    ^ C    ]ö  	Aj) 7  	 	) 7      A  *C    ^  *C    ]  - ¹A áA,ÝC   
  	Aj í!  ($!   6$@ E
  îA,âC  ($! ïE
  A: ¸  A: ¸Aè ÝC  
  ú!  ( !   6  E
 ûAè âCA  	(( 	( "
k"k  C    ]"E
 A  	(, 	($"k"k  C    ^"E
  	AjAj"
 Aj) "7  ) ! 
 § 
k6  	 7 	 § 
k6 	 	( k6 	 	( k6      A A A   - ¹A á  A: ¸A,ÝC     	Aj í!  ($!   6$@ E
  îA,âC  ($! ï E
  ("E
  Aj"6 
   ( (   	A0j$    ”  A(jß  ($!  A 6$@ E
  îA,âC  ( !  A 6 @ E
  ûAè âC  ( !  B 7 @@ E
  ("E
  Aj"6 
   ( (     «}# Ak"$ A !@@@@  - ¸Aj   ($ ñ!A!  (  ü
 @  ( ‚"E
  Aj Å@ (E
   *°!@ - AqE
 @ C  €?[
   A·j" -   C  C”•lAÿn:    ( !  ( "(0! (,!  (Aj"6 E
    ( (   (´A A A   (  - ¹Ö  Ò  ( !  ( "(0! (,!  (Aj"6 E
    ( ( A A A   (  - ¹Õ (" E
   Aj" 6  
   ( (  A ! Aj$        A     6   6   '  (!  A 6@ E
  „A¼âC  –  (!  A 6@@ E
  ("E
  Aj"6 
   ( (  @  ("E
  !@   ("F
 @ A|j"( ! A 6 @ E
  ¾AâC  G
   (!   6   ( kâC  (!  A 6@ E
  ¾AâC  (!  A 6@ E
  ("E
  Aj"6 
   ( (    †    ‹A âC A  A {A !@@@@@@ Aj   ((  (- @  (/"A€qE
 A?@ A€q
 A7 AÿqAF
 A÷    ((! œ# Ak"$ A ! A 6@  ("E
  AÝC ½"6@@  ("  (O
   6    Aj6  Aj Aj‘! (!   6 A 6 E
  ¾AâC Aj$ å@@  (  ( "kAu"Aj"A€€€€O
 A !@  ( k"Au"   KAÿÿÿÿ AüÿÿÿI"E
  A€€€€O
 AtÝC! ( ! A 6   Atj" 6   Atj! Aj!@  ("  ( "F
 @ A|j"( ! A 6  A|j" 6   G
   (!  ( !   6   6   (!   6@  F
 @ A|j"( ! A 6 @ E
  ¾AâC  G
 @ E
    kâC Å6 ð Ñ  (!  A 6@ E
  ¾AâC@  (  ("F
  A|j"( !@ E
  E
AÝC ½!  (!   6 E
 ¾AâC A 6   (!   6@ E
  ¾AâC  (A|j"( ! A 6 @ E
  ¾AâC   6˜# A0k"$  (P! (T!  )H7   Aj6,  Aj6( A j  (AjõA,ÝC¿" (Aj"6@ E
  ($!@ ((" ( "L
  (," L
    k  kAˆÀE
 Aj Å (!A !	@ ("
E
  (! ( !A !A J  Au A 
klqj! 
AW!	@@ 
Aq"
  	! 
! 	! 
!@  6   j! Aj! Aj! Aj" G
  
AI
 @  6  Aj  j"6  Aj  j"6  Aj  j"6  Aj  j"6  Aj  j"6  Aj  j"6  Aj  j"6  A j!  j! Axj"
  ($!
 ( ! A 6 B 7 Aðÿÿÿ6 B 7  - !@ (|AG
 @ - AG
   (h (l” (|AG
  (` (dæ$ A6| ç$@ (E
  Aq!  (L6”A !@ (P (H"kAj"E
 A JA J  AW6 AW!  6  6  6  6 Aðÿÿÿ6  Aj A G•E
  
Aj! Aj!@ ( 
k" J A Hr! ( ("kAu!
 	 Atj!@ "Aj". " k!@@ A
j. "AH
  
 Aj( !@ AJ
   j"AH
  k!A !@  j L
   k"AþÿÿÿK
 Aj! (  j!@Aÿ!@ -  AÿlAÿjAv"AÿF
  -  "  AsAÿqlAvj!  :   Aj! Aj! Aj"
     As j"  H" J    J"A HrrAq
     H" A  A J"k"Aj! (  j!@ Aj( -  AÿlAÿjAv"AÿF
 @@  jAqE
  !  -  "  AsAÿqlAvj:   Aj!  F
@  -  "  AsAÿqlAvj:   Aj" -  "  AsAÿqlAvj:   Aj! A~j"
   Aÿ Ø6 
Aj"

   Aj A G•
  (J (J 	J ($! ( !  (   Á A0j$  Ü# A k"$ @@  (x   (ŒJAt   (ˆJr   (€HAtr   („HAtr"G
  
@  (|
     å$  A6|   6d   6`    æ$  A6|  (p  (t    A€j Aj ´"E
  Aj! ( ! (!@@  (|E
     æ$    å$   6d   6`  A6| E
  Aj! !@   Aj"(  Aj"( æ$  A6| Aj"
    6t   6p   6x A j$ # Ak"$ @@  (”  (TL
 A !@@ Aðÿÿÿ6  (6  (,  (”  (LkAtj"( ! Aj( !  ( ! A 6@ E
   Atj!@ ( "( !	  (6 Aj (é$E
 Aj!A !@@@ E
 A !@@@ Aj"( "( "
 	F
  !@@ AqE
 A!A! Aj (é$E
 A ! Aj (é$E
 Aj"
 A ! Aq
 
 	F!A!@ (E
 @AÿA A€ (A	è$ (kA	u" Au"s k"
Aÿq"k  A€K 
  (\AF"Aÿ K  "
E
  ( 	 ( k"j 
Aÿ 
AÿI:   (!
@@  (AjG
  
 
/Aj;  
Aj"
6 
 (  j;  (A; ( ( j6  6 	Aj!	@ E
 A ! (  	L
 AÿA A€ (A	è$A	u" Au"s k"Aÿq"k  A€K   (\AF"Aÿ K  "E
  ( 	 ( k"j Aÿ AÿI (  	k"Ø6 (!@@  (AjG
   / j;  Aj"6  (  j;  ( ; ( ( j6   jAj6 
  ( (G
    (”Aj"6”   (TL
 A !   (”"6A!   Aj6” Aj$  Š}# AÐk"$    /  ;@  (
   ("(! (!AÝC  ¼!  (!   6 E
  ¾AâC A¼j  ´	@@ - ÌAG
   ("(! (! B 7  ²8  ²8 A¼j Ajû Aj A¼jý  ( Aj¿ A¤j  — A¤jÞ$ Ajà$"A :  A 6| A 6d B 7\  ("(! (! B 7€@@ ²C  €C”"‹C   O]E
  ¨!A€€€€x!  6Œ@@ ²C  €C”"‹C   O]E
  ¨!A€€€€x!  6ˆ@ AJ
  A 6ˆ  6€@ AJ
  A 6Œ  6„ A:  A¤jA Ý$@ (¸" (¤O
 @  Aj6¸ (´ AvAt"j(  Aÿq"j-  "Aÿ qE
  (° j(  Atj"*  Aj*  ˜ (¸" (¤I
   -  AG6\   “ ß$ A¤jÚ$ AÐj$ Aü~}}~~}}}}}}# AÀ k"$   Û$!  (" ( "kAm!@@  F
  Aj"Aj!A !@   Alj") "	78@ E
    A8j˜	 ) !	 C  úÆC  úF 	§¾"
 
C  úF^ 
C  úÆ]"88 C  úÆC  úF 	B ˆ§¾"
 
C  úF^ 
C  úÆ]"
8<@@@@ - " @  ( "Av"  (I
    Ü$  ( !  ( At"j( !
  ( j(  AÿqjA:   
 AtAøqj"Aj 
8   8     ( Aj6 @ E
  A|j-  AG
  A}j-  Aq
 @ Aj" F
   O
  Alj"- AG
 - 	Aq
 *  Atj* \
  * Axj* \
   C  €?’"88@  ( "Av"  (I
    Ü$  ( !  ( At"j( !
  ( j(  AÿqjA:   
 AtAøqj"Aj 
8   8     ( Aj6  E
  AG
  Aj" O
   Atj) "70 Aj" O
   Alj) "7(   Alj) "	7 @@ 
  B ˆ§¾! §¾! B ˆ§¾! §¾!   A0j˜	  ) 70   A(j˜	  ) 7(   A j˜	 ) !	 *<!
 *8! *,! *(! *4! *0! C  úÆC  úF  C  úF^ C  úÆ]"84 C  úÆC  úF  C  úF^ C  úÆ]"80 C  úÆC  úF  C  úF^ C  úÆ]"8, C  úÆC  úF  C  úF^ C  úÆ]"8( C  úÆC  úF 	§¾" C  úF^ C  úÆ]"8  C  úÆC  úF 	B ˆ§¾" C  úF^ C  úÆ]"8$ A 6  AjB 7  B 7  AÀ 6     
    ×$ A 6@ (E
   ( !A !@  Aj6 ( AvAüÿÿÿ qj(  A?qAlj"*!
 * ! !@ Av"
  (I
    
Ü$  ( !  ( 
At"
j( !  ( 
j(  AÿqjAA  r:    AtAøqj"Aj 
8   8     ( Aj"6  (" (I
 @ ("E
   Aj6 ( Atj!@ A|j"( J  ("Aj6 
  (J !  O
@  AljA	j-  AG
   Þ$ Aj" I
  AÀ j$  Ý@@ AÏ~qAÏ G
   (|AG
@  - AG
     (h  (l”  (|AG
    (`  (dæ$  A6|@ Aÿ~q"AG
 @@ C  €C”"‹C   O]E
  ¨!A€€€€x!@ C  €C”"‹C   O]E
    ¨ ³  A€€€€x ³ AjA
K
   - !@@ C  €C”"‹C   O]E
  ¨!A€€€€x! AG!@@ C  €C”"‹C   O]E
  ¨!A€€€€x!@ 
     ”  (|E
     æ$  A6|‚}# A°k"$ @  (
   ("(! (!AÝC  ¼!  (!   6 E
  ¾AâC A˜j A — à$"A :  A 6| A 6d B 7\  ("(! (! B 7€@@ ²C  €C”"‹C   O]E
  ¨!A€€€€x!  6Œ@@ ²C  €C”"‹C   O]E
  ¨!A€€€€x!  6ˆ@ AJ
  A 6ˆ  6€@ AJ
  A 6Œ  6„ A:   A˜j  C  €?š A 6\   “ ß$ A˜jÚ$ A°j$ A¼
}}}# AÐk"$  -  "AF! AF!AA - "AF!	 AF!  *”!
@@ 
 C  €?!C  €? ’	 “	’C   ?”•!A  !A 	 !
  
 
 ]!@@ ( (F
   6tA ! Aô jAjê$!	 A 6À@ ( ("kAu"AjAM
 @CÍÌÌ=  Atj* "
 
C½7†5_"!
@ AtAr" F
   Atj* !
 	  ”‹ C     
 
C    ]”‹ë$ Aj" ( ("kAu"AjAvI
  	  *”ì$  Aô j6 AjAjô$  
6D A 6d  6@ *!
  C   ?”80  
84   Aj A ›@ ( "E
   Aj6  (( Atj!@ A|j"( J  ( "Aj6  
  ((J@ ("E
   Aj6 ( Atj!@ A|j"( J  ("Aj6 
  (J ( "E
  Aj6  (¨ Atj!@ A|j"( J  ( "Aj6  
  (¨J  6t Aø jô$  
6´ A 6Ô  6° *!
  C   ?”8   
8¤   Aô j A œ@ ("E
   Aj6 (˜ Atj!@ A|j"( J  ("Aj6 
  (˜J (|"E
   Aj6| („ Atj!@ A|j"( J  (|"Aj6| 
  („J AÐj$ Ã}}# A k"$  ( "(  Ý$ A 6Ì A 6`@  Aj Aj°"Aÿ~qE
 @ *! *!@ E
   8  8 Aj  Aj˜	  *"8  *"8     ˜  Aj Aj°"Aÿ~q
  A j$ ´}}# A k"$  (  Ý$ A 6`@  Aj Aj±"Aÿ~qE
 @ *! *!@ E
   8  8 Aj  Aj˜	  *"8  *"8     ˜  Aj Aj±"Aÿ~q
  A j$  A   ( Ò   ( Ð   ( ÉÝ# Aà k"$ A !@@@ E
 A !  ("E
   (Aj"6 ! E
  (!  - ! A"jA :   A ;   Av6 A : $ !@ AG
  AvAÿq A€þƒxqr AtA€€üqr!  : -  : ,  6(@@ E
  A0j" )7  Aj Aj) 7    ()78 B 70  6@@ E
   (Aj"6 E
A !A !@ E
 A ! -  AG
 A ! ("E
   (Aj"	6 ! 	E
  6D   ("6H@ E
   (Aj"6 E
 (H!  6LA©!@@ /"AÿqAj       @@ Ahj	  A G
Aª!A«! A 6T  6P@@ /"AÿqAG
 @ A€qE
 @ - $AF
  A: $ Aÿ6  AvAÿqA;l AÿqAlj AvAÿqAljAä n!@ - $AF
  A: $  6   ›	 - $! A j"Aj Aj-  :    /  ;   E
  A : $ A 6 B 7 Aðÿÿÿ6 B 7   - !@ (|AG
 @ - AG
   (h (l” (|AG
  (` (dæ$ A6| ç$@ (E
  Aq!  (L6” (!@ (P (H"kAj"  (M
  J (J   AW6  AW!   6  6  6  6  Aðÿÿÿ6   A G"
•E
 @@ (" (4H
   (<N
  AØ j (HÅ (H"(  l"  (\K
 (X!@@ (@"
 A ! AØ j Å (@(  l" (\K
 (X j! (H! /"AøqE
   j!
 A€qA G AvAq"	AFq! 	A}j! ( ("kAu!@ " A
j". "AH
   Aj". " 	l"jA  !@@ (D"
 A ! (0! AØ j Å (X (D(   (4klj j (0"k! / ! 
 j!  kA   J! Á" (8" k  j H!@@ E
  	E
  Aj( !   	l"j!   j!@ - -AG
 @ E
  - $
	  N
@ (!@ E
    j-  lAÿm! Aj   Aj-  "j  lA~mj":   Aÿl Aÿqm" - "lAÿ k"  -  ljAÿm!@@ - ,AG
   :   Aj  - !l   Aj-  ljAÿm:   Aj  -  l   Aj-  ljAÿm:    Aj-  !  Aj-  ! -  ! - !!   j"-  "Aÿs -  l  ljAÿm:   Aj" -  "Aÿs -  l  l  ljAÿm ljAÿm:   Aj" -  "Aÿs -  l  l  ljAÿm ljAÿm:    Aj!  Aj! Aj" G
   AK
 - $
  N
@ (!@ E
    j-  lAÿm!  Aj-  !  Aj-  ! -  ! - !!   j"-  "Aÿs -  l  - "lAÿ k"  -  ljAÿm ljAÿm:   Aj" -  "Aÿs -  l  l  ljAÿm ljAÿm:   Aj" -  "Aÿs -  l  l  ljAÿm ljAÿm:    	j!   	j!  Aj" G
  @ E
  - $
  N
@ (! @ E
     j-  lAÿm!     j-  "l" Aÿm!@  AþjAýI
 @ AÿG
  Aj :    -  :   Aj - !:   Aj - ":  @ Aj"-  "
   :    -  :   Aj - !:   Aj - ":    Aÿs" l  jAÿm:    -   l  -  ljAÿn:   Aj"  - ! l   -  ljAÿn:   Aj"  - " l   -  ljAÿn:   Aj! Aj" G
  @ AK
  - $
  N
@ (!@ E
    j-  lAÿm!  -  lAÿ k"  -  ljAÿm!@@ - ,AG
   :   Aj  - !l   Aj-  ljAÿm:   Aj  - "l   Aj-  ljAÿm:    Aj-  !  Aj-  ! - "! - !!   j"-  "Aÿs -  l  ljAÿm:   Aj" -  "Aÿs -  l  l  ljAÿm ljAÿm:   Aj" -  "Aÿs -  l  l  ljAÿm ljAÿm:     	j!   	j! Aj" G
   	AG
 - $AG
  N
 ( !@ (!@ E
    j-  lAÿm!Aÿ k  -  l  lj!@@ - ,E
  !  j-  "Aÿs -  l Aÿm lj! Aj!  Aÿm:    Aj!  ! Aj" G
   Aj (T"Auj! (P!@ AqE
  (  j( !   	    Aj(     Aj"
    
•
  (J (J (H! B 7H@ E
  (" E
   Aj" 6  
   ( (   (D! A 6D@ E
  (" E
   Aj" 6  
   ( (   (@! A 6@@ E
  (" E
   Aj" 6  
   ( (  @ E
  ("E
  Aj"6 
   ( (   Aà j$  ¯ µ@  - AG
 @  N
   (!  j!@  (   j-  l!@@ E
    j-  lAüm! Aÿm!@ E
  !	@ AÿF
 Aÿ k -  l  ljAÿm!	  	:   Aj! Aj" G
 ¯ Á@@@  - 
    lj!  - 
  N
@  ( !@@  - AG
  E
   j-  lAÿm!   j-  l!@ E
    j-  lAüm! Aÿm! !@ E
 @ AÿG
    (6  !@@ Aj"	-  "
  	 :     - :   Aj  - :   Aj  - :   	  j  lA~mj":    Aÿl Aÿqm"  - lAÿ k" -  ljAÿm:   Aj"	   - l  	-  ljAÿm:   Aj"	   - l  	-  ljAÿm:  A!  j! Aj" G
  ¯   N
 @  ( !@@  - AG
  E
   j-  lAÿm!   j-  l!@ E
    j-  lAüm! Aÿm!@ E
 @ AÿG
    (6  Aj"  -  "j  lA~mj":    Aÿl Aÿqm"  - lAÿ k" -  ljAÿm:   Aj"   - l  -  ljAÿm:   Aj"   - l  -  ljAÿm:   Aj! Aj" G
 …@@@  - 
    lj!  - 
  N
 A}j!@  ( !@@  - AG
  E
   j-  lAÿm!   j-  l!@ E
    j-  lAüm! Aÿm!@@ E
 @ AÿG
 @@     (6    - :   Aj  - :   Aj  - :   Aj!    - lAÿ k"	 -  ljAÿm:   Aj"
   - l 	 
-  ljAÿm:   Aj"
   - l 	 
-  ljAÿm:    j! Aj" G
  ¯   N
  A}j!@  (   j-  l!@@ E
    j-  lAüm! Aÿm!@@ E
 @ AÿG
 @@     (6    - :   Aj  - :   Aj  - :   Aj!    - lAÿ k"	 -  ljAÿm:   Aj"
   - l 	 
-  ljAÿm:   Aj"
   - l 	 
-  ljAÿm:    j! Aj" G
 ä}}}}}}}}}# Aàk"$  A0j  (Å@ (4E
    /  ;@ E
  -  AÿqE
  AÈj  — A0jà$"A :  A 6| A 6d B 7\  ("	(!
 	(!	 B 7€@@ 	²C  €C”"‹C   O]E
  ¨!	A€€€€x!	  	6Œ@@ 
²C  €C”"‹C   O]E
  ¨!
A€€€€x!
  
6ˆ@ 
AJ
  A 6ˆ  
6€@ 	AJ
  A 6Œ  	6„ A:  AÈjA Ý$@ (Ü"	 (ÈO
 @  	Aj6Ü (Ø 	AvAt"
j(  	Aÿq"	j-  "Aÿ qE
  (Ô 
j(  	Atj"	*  	Aj*  ˜ (Ü"	 (ÈI
   -  AG6\     - AqAvA ¡ ß$ AÈjÚ$ E
  A€€€I
 @ , AJ
  AÈj  — A0jà$"	A :  	A 6| 	A 6d 	B 7\  ("(!
 (! 	B 7€@@ ²C  €C”"‹C   O]E
  ¨!A€€€€x! 	 6Œ@@ 
²C  €C”"‹C   O]E
  ¨!
A€€€€x!
 	 
6ˆ@ 
AJ
  	A 6ˆ 	 
6€@ AJ
  	A 6Œ 	 6„ 	A:  	 AÈjA  C  €?š   	  - AqAv  - ¡ 	ß$ AÈjÚ$ B 7Ø B€€€€€€€À?7Ð B€€€ü7È B 7( B€€€€€€€À?7  B€€€ü7@ E
  *!
 *! * ! *! A 6,   ‹" ‹"  ]"•8$  
 •8    •8   •8 A0j Aj‹	 *! *! * ! *!
  *" *4"” *<" *"”’8Ô   *0"”  *8"”’8Ð   ” 
 ”’8Ì   ” 
 ”’8È   ”  ”’ *D’8Ü  *@  ”  ”’’8Ø   AÈj— A0jà$"	A :  	A 6| 	A 6d 	B 7\  ("(!
 (! 	B 7€@@ ²C  €C”"‹C   O]E
  ¨!A€€€€x! 	 6Œ@@ 
²C  €C”"‹C   O]E
  ¨!
A€€€€x!
 	 
6ˆ@ 
AJ
  	A 6ˆ 	 
6€@ AJ
  	A 6Œ 	 6„ 	A:  	  Aj  *Èš   	  - AqAv  - ¡ 	ß$ Ú$ Aàj$ A×# A0k"$  A j  (Å@@ ($E
 @@  ("E
  Aj! Aj! (! (!  ("Aj! Aj!A !A ! ( !  ( 6  6  6  6   õ (" ( "L
  (" ("L
 @@  ("	E
  	-  
@  - AG
  A€€€I
  6  6  6  6   ()7( B 7  Aÿq!
 Av"Aÿq! Av"Aÿq! Aj A jõ ("
 ("k!  (/!@ Av"AÿG
 @ AøqA G
  (" (N
 At 
Atr rA€€€xr! Aq! Aÿÿÿÿq! AtAu"	AjAI!
@ A j  (" Æ ( "Aq
 (" ($AvK
  ("I
   kK
@ E
   Atj!A ! 	!@ E
 @  6  Aj! Aj! Aj" G
  

 @  6  Aj 6  Aj 6  Aj 6  Aj 6  Aj 6  Aj 6  Aj 6  A j! Axj"
  Aj" (H
   (" (N
 Aq! 
Al A}ljA«ÕªÕzlAjAK!
@ A j  (" Æ (" ($AnK
  ("	I
   	kK
@ 
 F
 A ! (  	Alj"	!@ E
 @  :   :   :   Aj! Aj" G
  
E
  	 Alj!@  :   :   :   Aj :   Aj :   Aj :   Aj :   Aj :   Aj :   Aj :   Aj :   Aj :   Aj :   A
j :   Aj :   Aj :   A
j :   A	j :   Aj :   Aj :   Aj :   Aj :   Aj :   Aj :   Aj" G
  Aj" (H
  @ A€qE
  (" (N
 Aÿÿÿÿq! Aÿl!
@ A j  (" Æ (" ($AvK
  ("I
   kK
@ E
  (  Atj" Atj!	@@@ - "
   :   :   :   :   Aÿ 
  j  lAÿnkAÿqn"k" -  l  ljAÿm:     - l  ljAÿm:    - l  
ljAÿm:  Aj" 	G
  Aj" (H
   (! (!	@ AøqA G
  	 N
 Aÿÿÿÿq!
  
l!  l!  l! Aÿs!@ A j  (" 	Æ (" ($AvK
  ("I
   kK
@ 
E
  (  Atj" Atj!@   -  l jAÿn:     - l jAÿn:    - l jAÿn:  Aj" G
  	Aj"	 (H
   	 N
  
l!  l!  l! Aÿs!@ A j  (" 	Æ (" ($AnK
  ("I
   kK
@ 
 F
  (  Alj" Alj!@   -  l jAÿn:     - l jAÿn:    - l jAÿn:  Aj" G
  	Aj"	 (H
    (    k  k Ø  (! !
@ 	("	E
  	 	(Aj"
6 
E
 (!
     k  k 	   k 
 kA A   - Ö A0j$ A @@ ("E
    )7   Aj Aj) 7    ()7  B 7 ¢# A0k"$    (Å@@@ (
 A!  (! (!  6   j6  6    j6@@  ("E
 @  ä"
 A!  (! (!@  ("E
   (Aj"	6 	E
 A A    A A A A A Õ  ( ä"
 A!  ( (k! ( ( k! Au q! Au q!@  - AG
   6$  6   6  6 A 6 A 6@  Aj Aj A$j A j ( ( Aj AjA ëE
 @ /"  /"G
 @  AÿqA G
  ( AH
A !@ A(j  ( jÆ ("  (,AvK
   ("I
 ((!	 A(j  ( j ( (  (" (,AvK
  ("I
 ($"  kK
    kK
@ E
  	 Atj!  (( Atj! AjAÿÿÿÿq!@@ Aq
  !   - :     - :    -  :    - :   Aj!  Aj! E
   Atj!@   - :     - :    -  :    - :   Aj Aj-  :    Aj Aj-  :    Aj Aj-  :    Aj Aj-  :    Aj!  Aj" G
  Aj" ( H
    AG
 ( AH
A !@ A(j  ( jÆ ("  (,AnK
   ("I
 ((!	 A(j  ( j ( (  (" (,AnK
  ("I
 ($"  kK
    kK
@ E
  	 Alj! A ! (( Alj"	!@ Aq"E
 @   - :     - :    -  :   Aj!  Aj! Aj" G
  AM
  	 Alj!@   - :     - :    -  :   Aj Aj-  :    Aj Aj-  :    Aj Aj-  :    Aj Aj-  :    Aj Aj-  :    Aj Aj-  :    A	j Aj-  :    A
j A
j-  :    Aj A	j-  :    Aj!  Aj" G
  Aj" ( H
  @  AG
  A G
 ( AH
A !@ A(j  ( jÆ ("  (,AnK
   ("I
 ((!	 A(j  ( j ( (  (" (,AvK
  ("I
 ($"  kK
    kK
@ AÿÿÿÿqE
  	 Alj!  AjAÿÿÿÿq!	A ! (( Atj"
!@ Aq"E
 @   - :     - :    -  :   Aj!  Aj! Aj" G
  	AM
  
 Atj!@   - :     - :    -  :   Aj Aj-  :    Aj Aj-  :    Aj Aj-  :    Aj A
j-  :    Aj A	j-  :    Aj Aj-  :    A	j Aj-  :    A
j A
j-  :    Aj Aj-  :    Aj!  Aj" G
  Aj" ( H
    A€rA G
@@ Ahj	  ( AH
A !	@ A(j  ( 	jÆ ("  (,AvK
   ("I
 ((! A(j  ( 	j ( (  (" (,AnK
  ("I
 ($"  kK
    kK
@ E
   Atj! A ! (( Alj"
!@ Aq"E
 @   - :     - :  -  !  Aÿ:    :   Aj!  Aj! Aj" G
  AM
  
 Alj!@   - :     - :  -  !  Aÿ:    :   Aj Aj-  :    Aj Aj-  :   Aj-  !  AjAÿ:    Aj :    Aj Aj-  :    A	j Aj-  :   Aj-  !  AjAÿ:    A
j :    Aj Aj-  :    A
j A
j-  :   A	j-  !  AjAÿ:    Aj :    Aj!  Aj" G
  	Aj"	 ( H
    A G
 ( AH
 A !	@ A(j  ( 	jÆ ("  (,AvK
   ("I
 ((! A(j  ( 	j ( (  (" (,AvK
  ("I
 ($"  kK
    kK
@ E
   Atj!  AjAÿÿÿÿq!
A ! (( Atj"!@ Aq"E
 @   - :     - :  -  !  Aÿ:    :   Aj!  Aj! Aj" G
  
AM
   Atj!@   - :     - :  -  !  Aÿ:    :   Aj Aj-  :    Aj Aj-  :   Aj-  !  AjAÿ:    Aj :    Aj A
j-  :    A	j A	j-  :   Aj-  !  AjAÿ:    A
j :    Aj Aj-  :    A
j A
j-  :   Aj-  !  AjAÿ:    Aj :    Aj!  Aj" G
  	Aj"	 ( H
  (" E
   Aj" 6@  
   ( (   (" E
   Aj" 6@  
   ( (  A! A !      Ê! @ E
  ("E
  Aj"6 
   ( (   A0j$    '@  (" E
     (Aj"6 
    ó# Ak"$  Aj  (Å@@@ (E
  ( ("k!	 ( ( "k!
  - !  (!  (! @ - AqE
      
 	       AqÖ!      
 	      AqÕ! A!  E
  ("E
  Aj"6 
   ( (   Aj$    Ã# AÐk"
$  
A8j  (Å@@@ 
(<E
 @  (G
   (G
  
 6D 
 6@ 
B 78     
A8j   	ª!  
 6À 
AÀjAj"  j6  
 6Ä 
  j6Ì 
AÀjô 
A°jAj" ) 7  
 
)À7° 
A°j õ 
A8jÞ"  Aj  (C  €?  
A°jA A A   -  	á  (  
(À" k6  
 
(°  k6° 
 
(´ 
(Ä" k6´ 
 
(¼  k6¼@ 
Aj     
A°j í"ïE
  A ñ î ßA! A!  E
  ("E
  Aj"6 
   ( (   
AÐj$    œ# Ak"$  Aj (Å@@@ (E
  (!	  AA¼ÝC Aj 	      - ƒ‰  AA ‰ E
  ("E
  Aj"6 
   ( (   Aj$  ?# Ak"$  Aj  (Å@@ (
 A!   …!  Aj$   ‹@ E
   (Aj"6 E
    ˜A ÝC"B 7  6 A´Ð"6   6  :   :  AjB 7  AjA ;  /"AF
  AF
    ”A †A,ÝC¿" (Aj"6@ E
     À! (!@ E
   Aj"6 E
   ˜A ÝC"B 7  6 A´Ð"6   6 AjB 7  AjA 6  /"AF
 AF
   ”  E
   Aj"6@ 
   ( (  @ E
  ("E
  Aj"6 
   ( (    Ä}  Aj!  Aì j!  Aè j!@@@@@@  (`   (d!  (   ²!  A6`   6d@ Aÿ~q
 A ! õ$  *  * Aö$@@@  (   ²"Aÿ~q"AjA
K
    6d * !@ AG
   8   * 8    *  ö$@ 
   A 6d AqAG
   *  *  ö$ A ø$  A6`   ú$"Aÿ~q
  A6`   ˆ}  Aj!@@@@@@  (`   (d!A !@  ( "(" ( O
   Aj6   ( AvAüÿÿq"j(  Aÿq"Atj"* 8h   Aj* 8l ( j(  j-  !  A6`   6d@ Aÿ~q
 A ! õ$   *h  *lAö$@@  ( "(" ( O
 @  Aj6  ( AvAüÿÿq"j(  Aÿq"Atj"* 8   Aj* "8 @@@ ( j(  j-  "Aÿ q"AjA
K
    6d * ! AG
   8h   * 8l E
 AqAG
  *   ö$   *  ö$  ( "(" ( I
   A 6d A ø$  A6`   ú$"Aÿ~q
  A6`   –}  Aj!@@@@@@  (Ì   (Ð!A !@  ( "(" ( O
   Aj6   ( AvAüÿÿq"j(  Aÿq"Atj"* 8Ô   Aj* 8Ø ( j(  j-  !  A6Ì   6Ð@ Aÿ~q
 A ! í$   *Ô  *ØAî$@@  ( "(" ( O
 @  Aj6  ( AvAüÿÿq"j(  Aÿq"Atj"* 8   Aj* "8 @@@ ( j(  j-  "Aÿ q"AjA
K
    6Ð * ! AG
   8Ô   * 8Ø E
 AqAG
  *   î$   *  î$  ( "(" ( I
   A 6Ð A ð$  A6Ì   ó$"Aÿ~q
  A6Ì    @@@  - AG
 @@  - XAG
   á$  (|AG
 @  - AG
     (h  (l”  (|AG
    (`  (dæ$   6p   6h  A 6|   6t   6l     (ŒJAt   (ˆJr   (€HAtr   („HAtr"6x 
    å$  A6|@  (|AG
     (`  (dæ$  A6|    å$  A6|   6d   6`œ}}}}}}}}}}}}}}A !@ ²  ²"“"	¼AÿÿÿÿqAÿÿÿûJ
  ² ²"
“"¼AÿÿÿÿqAÿÿÿûJ
  ( " ("
C`B¢C`B¢
   J 	 	C    ["C    ^"²" “ •"	 ("  ("C`B¢C`B¢
   J  C    ["C    ^"²" 
“ •" 	 ]""C  €?_E
   	 !A !@ C    ^E
 @@ ‹C   O]E
  ¨!A€€€€x!  6 @@ ‹C   O]E
  ¨!A€€€€x!  6  Aj! Aj!A! C  €?_E
  
  ²" “ •"    ²" 
“ •"  ]" !@ C    ^
  C    ^E
@  _E
 @ C    ^E
 @@   ” ’ 	 ^""‹C   O]E
  ¨!A€€€€x!  6 @@  	” 
’  "	‹C   O]E
  	¨!A€€€€x!  6  Aj! Aj! Aj!@@ C  €?]E
 @  E
 @@ ‹C   O]E
  ¨! A€€€€x!    6 @  ” 
’"	‹C   O]E
  	¨!A€€€€x!@@  ” ’"	‹C   O]E
  	¨! A€€€€x!    6 @ ‹C   O]E
  ¨!A€€€€x!  6   6  Aj@@   	 ^" "	‹C   O]E
  	¨!A€€€€x!  6 @@    "	‹C   O]E
  	¨! A€€€€x!    6  Aj!  A    " A˜Ñ"6         A0âC    A A A ®     A A ®    A   ®      A ¯m@ 
      A ¯  (Aj"6@ E
       ¯! ("E
   Aj"6@ 
   ( (       (,      Û" A : (  A 6$  A¨Ñ"6         A A ÁÞ~# Ak"$   A$j!@@  - ("AÿF
 @ 
  ( E
 A 6  A )ÀÑ"7 Aj  Aj Atj(    A : (  A 6$A !  A 6   B 7   ;@ AH
  AH
  Aÿq"E
 @@ 
  Aj  È - AG
 (! Aj A Ç - AG
  (I
 ­ ­~"	BÿÿÿÿV
 @@ E
 @  - ("AÿF
 @ 
  (  F
  6  A )ÀÑ"7 Aj  Aj Atj(    A : (   6$ 	BûÿÿÿV
 	§AjAH!@@@  - ("AÿF
 @ AG
  ( !  6  E
 J  - ("AÿG
¯  A )ÀÑ"7 Aj  Aj Atj(    A: (   6$A!  Aj6A ! A )ÐÑ"7 Aj  Aj Atj(  E
   6    6   6A! Aj$  ˜~# A k"$ @@  - ("AÿF
   Aj6A ! A )ÐÑ""7@@@ Aj  A$j" Aj Atj(  
 A !   ( ( /A A ÁE
 (!  ("6   kAu6  )7    ãA! (AH
A !@  - ("AÿF
  Aj6  7 Aj  Aj Atj(  !  ( ! Aj   ( ( @  ( "E
    lj ( Ö6A! Aj" (H
   E
 (" E
   Aj" 6  
   ( (   A j$  ¯ ^# Ak"$ @  - ("AÿF
  A )ÀÑ"7 Aj  A$j Aj Atj(    Aÿ: (  Ü!  Aj$   _# Ak"$ @  - ("AÿF
  A )ÀÑ"7 Aj  A$j Aj Atj(    Aÿ: (  ÜA,âC Aj$ Ì~# Ak"$ @ - ("AÿF
   Aj6 A )ÐÑ""7@@ Aj A$j" Aj Atj(  
   B 7  - ("AÿF
  Aj6  7 Aj  Aj Atj(  ! ( ! (!   6     l6 Aj$ ¯ î~# Ak"$ @@ - ("AÿF
   Aj6 A )ÐÑ""7@@@ Aj A$j" Aj Atj(  E
  - ("AÿF
  Aj6  7 Aj  Aj Atj(  ! ( " (l"
  B 7    l"I
   kK
   6    j6  Aj$ ¯ Ï~# Ak"$   ß!@@  - ("AÿF
   Aj6 A )ÐÑ""7@ Aj  A$j" Aj Atj(  E
   - ("AÿF
  Aj6  7 Aj  Aj Atj(    (   (" l"E
   AL
  j! Aj$  ¯ «# Ak"$   A$j! ( "- (!@@@  - ("AÿG
  AÿF
 AÿG
  A )ÀÑ"7 Aj  Aj Atj(    Aÿ: (  6 A )ÈÑ"7 Aj  A$j Aj Atj(   ( !@  ("E
    6 J  A 6  B 7   (6   (6   (6 A 6 B 7@@ ( "- ("AÿF
  A$j!@ 
  ( E
 A 6  A )ÀÑ"7 Aj  Aj Atj(   A : ( A 6$   ( /;   ( (6   ( (6   ( ( 6  Aj$ ä~# Ak"$ @@@@  - ("AÿF
   Aj6 A )ÐÑ""7 Aj  A$j" Aj Atj(  E
  - ("AÿF
   Aj6  7 Aj  Aj Atj(  !  (   ("l"E
@@@@@@@  /"Aj 



























 @@ Aÿ}j  A F

 AA  AÿÿÿK Ø6	 AA    ê Ø6  Av Ø6    ê Ø6 Aj ›	@ - " - "G
   - AÿqF
  (AH
A !@ Aj   Æ  (" (AnK
@ E
  (!A ! !@ Aq"E
 @  / ;   Aj AjAj-  :   Aj! Aj! Aj" G
  AI
 @  / ;   Aj AjAj"-  :   Aj -  :   Aj / ;   Aj -  :   Aj / ;   A	j / ;   Aj -  :   Aj! A|j"
  Aj"  (H
   A€€€xr µ!  (! AH
A !@ Aj   Æ ("Aq
  (" (AvK
@ E
  AtAu"Aj!	A !@ Aq"E
 @  6  Aj! Aj! Aj" G
  	AI
 @  6  Aj 6  Aj 6  Aj 6  Aj 6  Aj 6  Aj 6  Aj 6  A j! Axj"
  Aj"  (H
  ¯    Ø6  Aj$ ¹# A k"$   6  6  6  6 @@  - ("AÿF
   Aj6A ! A )ÐÑ"7@@ Aj  A$j Aj Atj(  
  ! A 6 A 6A!@   Aj Aj Aj Aj ( ( Aj A ë
  !@  /" /F
 A !    ( ( ( (  ( ( Ë!A! ( ! (! (! (! (!	 (!
@@ AÿqAF
    
 	     Ì   
 	     ÍA !@ E
  (" E
   Aj" 6  
   ( (   A j$  ¯ È~~# A0k"	$  	 6$@@@@  (  (F
 A !
A !
 A H
   - "­ ­~"BÿÿÿÿV
   - ("AÿF
 	 	A+j6,A ! 	A )ÐÑ""
7@@ 	A,j  A$j"
 	Aj Atj(  
   ( !A !   - ("AÿF
 	 	A+j6, 	 
7 	A,j 
 	Aj Atj(  !  ( "  (l!     l Bˆ§j"
I
 	   
k6 	  
j6 	 	)7 	AjAˆ  AF 	Aj    	A$j  ï 	(" 	(G
@ E
  	 6 JA!
 	($! 	A 6$@ E
  (" E
   Aj" 6  
   ( (   	A0j$  
¯ °~# Ak"$ @@@@@ AH
   A$j!	   /AvAq"
l!  
l!  
l!A !
A )ÐÑ"!
@  - ("AÿF
  Aj6  
7 Aj 	 Aj Atj(  !  ( ! Aj  
 j ( (   (K
@ E
    
 jlj j ( j Ö6 
Aj"
 G
   E
 ("
E
  
Aj"
6 

   ( (   Aj$  ¯ ë
~# Ak"$ @@ AH
   A$j!	A !
A )ÐÑ"!@  - ("AÿF
  Aj6  7 Aj 	 Aj Atj(  !
  ( ! Aj  
 j ( ( A !@ A L
  
  
 jlj! (!@   j"
Am"j" -  "A At 
kAj"
tr A~ 
wq   j"
Am"j-   At 
kAjvAq:   Aj" G
  
Aj"
 G
 @@ E
  ("E
  Aj"6 
   ( (   Aj$  ¯ ¥# Ak"$ @@  /A G
   - ("AÿF
  Aj6 A )ÐÑ"7 Aj  A$j Aj Atj(  E
 @  (AH
 A !@ Aj   Æ  (" (AvK
@ E
  AjAÿÿÿÿq!A ! ("!@ Aq"E
 @  - :  Aj! Aj" G
  AM
   Atj!@  - :  Aj Aj-  :   A
j Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   Aj Aj-  :   A j" G
  Aj"  (H
  Aj$  ¯ þ# Ak"$ @@  /A G
   - ("AÿF
  Aj6 A )ÐÑ"7 Aj  A$j Aj Atj(  E
 @  (AH
 A !@ Aj   Æ  (" (AvK
@ E
  AjAÿÿÿÿq!A ! ("!@ Aq"E
 @ Aÿ:  Aj! Aj" G
  AM
   Atj!@ Aÿ:  AjAÿ:   AjAÿ:   AjAÿ:   AjAÿ:   AjAÿ:   AjAÿ:   AjAÿ:   A j" G
  Aj"  (H
  Aj$  ¯ â# Ak"$ @@  ( (G
   ( (G
  /AˆG
   - ("AÿF
  Aj6 A )ÐÑ"7 Aj  A$j Aj Atj(  E
 @@@@  /"A F
  A G
A!  (AH
A !@ Aj   Æ  (" (AvK
 (! Aj  Æ  (" (K
A !@ A L
  (!@  F
  F
  AtjAj"	  j-   	-  lAÿn:   Aj"  (H
  Aj"  (H
  A !  A ÑE
A!  (AH
A !@ Aj   Æ  (" (AvK
 (! Aj  Æ  ("	 (K
A !@ 	A L
  (!@  	F
  F
  AtjAj  j-  :   Aj"  (H
  Aj"  (H
 A! E
 ("	E
  	Aj"	6 	
   ( (   Aj$   ¯ ´~# A0k"$ @@ AF
  AˆF
  A G
A!@   /"F
 @@ A F
  AˆG
 AG
  (  (G
  Aˆ; A G
   A ;  ÏA !  ("AH
   ("AH
  A8q"E
  A j  È - $AG
  ( "­ ­~"B€€€€Z
  BûÿÿÿV
  §Aj"AH"E
 @ A G
  Aÿ Ø6  ("AF
  6  6  (!  (!   6   Aj"6 E
  )7 A j  Aj    AjA A ï@  ("E
    6 J   ( 6   ($6   ((6 A 6( B 7  (! A 6@ E
  ("E
  Aj"6 
   ( (  @@  - ("AÿF
   A$j!@ AG
  ( !  6  E
 J A )ÀÑ"7  Aj  A j Atj(    A: (   6$   6    ;  ("E
   Aj"6A! 
     ( (   A0j$   ð	# Ak"$ @@ C    `E
  C  €?_E
 A!  - Aq
 @ C  €?[
   - ("AÿF
  Aj6A ! A )ÐÑ"7 Aj  A$j Aj Atj(  E
   A ÑE
 A!  (AH!@@ C  C”"‹C   O]E
  ¨!A€€€€x! 
 A !@ Aj   Æ  (" (AvK
@ E
  AjAÿÿÿÿq!A ! ("	!@ Aq"
E
 @  -  lAÿm:  Aj! Aj" 
G
  AM
  	 Atj!
@  -  lAÿm:  Aj" -   lAÿm:   Aj" -   lAÿm:   Aj" -   lAÿm:   Aj" 
G
 A! Aj"  (H
  Aj$   ¯ ò
~# Ak"$  Aÿq! Aÿq! E AÿÿÿFq! Av"Aÿq! Av"	Aÿq!
 AvAÿq! AvAÿq!@@@  - AK
 @ E
   (  (F
  æ  - "AF
A t"A AJ!  k!  
k!	  k!A !@  ( Atj"  ( "AvAÿqA;l AÿqAlj AvAÿqAljAä n"lAÿm j 	 lAÿm 
jAtr  lAÿm jAtrA€€€xr6  Aj" G
    (!@ 
  AH
  k!  
k!  k!  A$j!
A !A )ÐÑ"!@  - ("AÿF
  Aj6  7 Aj 
 Aj Atj(  !@  (AH
   /AvAq!
   (  lj!A !@ Aj"  Aj"-  A;l -  Alj -  AljAä n"lAÿm j:     lAÿm 	j:     lAÿm j:    
j! Aj"  (H
  Aj"  (H
   AH
   A$j!A !
A )ÐÑ"!@  - ("AÿF
  Aj6  7 Aj  Aj Atj(  !@  (AH
   /AvAq!   (  
lj!A !@ Aj" Aj"-  A;l -  Alj -  AljAä n":    :    :    j! Aj"  (H
  
Aj"
  (H
  Aj$ ¯ u# Ak"$ @  - ("AÿF
   Aj6 A )ÐÑ"7@ Aj  A$j Aj Atj(  E
   - Aq
     Ó Aj$ ¯ ²	# AÀk"$   6¬  6°  6¨  6¤  6   6œ@@ - Aq
   - ("AÿF
  A·j6¸A ! A )ÐÑ"7x@ A¸j  A$j Aø j Atj(  E
   - AøqE
 A!   A°j A¬j A¨j A¤j ( ( A j Aœj 	ëE
 A !@@ 	
 A !A !
A !A !A !
A ! 	-  AG
 @ 	("E
   (Aj"6 E
 	(! 	(!
 Aø jÃ! /!  /! (!  ("6p   kAu6t  )p78@    A8jA   
ÅE
   /!@ /"Aðq"
  ( (F
A! (¤AH
  AvAq! AvAq!A !@ A¸j   (¬ jÆ (¼" (° l"I
 (¸! A¸j  (œ j ( (  (¼"	 (  l"I
 (¸!
