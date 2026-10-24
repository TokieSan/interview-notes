# Processing Files Too Big for Memory

Shell and C++ techniques for the "what if the file is 500 GB?" follow-up: external sort, chunking, parallelism, `mmap`.

# Linux Commands

1. Sorting Large Files:
- `sort` with the `-S` option to specify memory buffer size and `-T` for temporary directory and `--parallel=4` for multiple threads:
```bash
sort -S 2G -T /tmp/sortdir --parallel=4 bigfile.txt > sorted_output.txt
```

- For files larger than available RAM, use external merge sort:
```bash
# Split into smaller chunks
split -l 1000000 hugefile.txt chunk_
# Sort each chunk
for f in chunk_*; do sort -o ${f}.sorted $f; done
# Merge sorted chunks
sort -m chunk_*.sorted > final_sorted.txt
```

2. Writing/Processing Large Files:
- Use `tee` to write while monitoring:
```bash
process_command | tee output.txt
```

3. Parallel Processing with `xargs` and `parallel`:
```bash
# Process files in parallel using xargs
find . -type f | xargs -P 4 -I {} process_file {}

# Using GNU parallel (more powerful)
find . -type f | parallel -j 4 process_file {}

# Parallel sort
sort --parallel=4 -S 2G bigfile.txt > sorted.txt

# Split and parallel process
split -n l/4 bigfile.txt chunk_
ls chunk_* | parallel -j 4 'sort {} > {}.sorted'
sort -m chunk_*.sorted > final_sorted.txt
```

2. Using `awk` with multiple processes:
```bash
# Process different portions of file in parallel
awk 'NR%4==0' bigfile.txt | process1 &
awk 'NR%4==1' bigfile.txt | process2 &
awk 'NR%4==2' bigfile.txt | process3 &
awk 'NR%4==3' bigfile.txt | process4 &
wait
```

# C++

```cpp
#include <fstream>
#include <iostream>
#include <string>
#include <vector>
#include <algorithm>

// Reading large files in chunks
void read_large_file(const std::string& filename) {
    std::ifstream file(filename, std::ios::binary);
    constexpr size_t CHUNK_SIZE = 1024 * 1024; // 1MB chunks
    std::vector<char> buffer(CHUNK_SIZE);

    while (file) {
        file.read(buffer.data(), CHUNK_SIZE);
        std::streamsize bytes_read = file.gcount();
        
        // Process your chunk here
        process_chunk(buffer.data(), bytes_read);
    }
}

// External merge sort implementation
void external_merge_sort(const std::string& input_file, const std::string& output_file) {
    constexpr size_t CHUNK_SIZE = 1024 * 1024 * 100; // 100MB chunks
    std::vector<std::string> chunk_files;

    // Phase 1: Split and sort chunks
    {
        std::ifstream input(input_file);
        std::vector<std::string> chunk;
        std::string line;
        size_t current_size = 0;
        int chunk_number = 0;

        while (std::getline(input, line)) {
            chunk.push_back(line);
            current_size += line.length();

            if (current_size >= CHUNK_SIZE) {
                // Sort and write chunk
                std::sort(chunk.begin(), chunk.end());
                std::string chunk_name = "chunk_" + std::to_string(chunk_number++) + ".txt";
                std::ofstream chunk_file(chunk_name);
                for (const auto& l : chunk) {
                    chunk_file << l << '\n';
                }
                chunk_files.push_back(chunk_name);
                chunk.clear();
                current_size = 0;
            }
        }

        // Handle last chunk
        if (!chunk.empty()) {
            std::sort(chunk.begin(), chunk.end());
            std::string chunk_name = "chunk_" + std::to_string(chunk_number) + ".txt";
            std::ofstream chunk_file(chunk_name);
            for (const auto& l : chunk) {
                chunk_file << l << '\n';
            }
            chunk_files.push_back(chunk_name);
        }
    }

    // Phase 2: Merge sorted chunks
    // (Implementation of k-way merge would go here)
}

// Writing large files efficiently
void write_large_file(const std::string& filename, const std::vector<std::string>& data) {
    std::ofstream file(filename, std::ios::binary);
    file.rdbuf()->pubsetbuf(nullptr, 0); // Disable buffering for large writes

    // Write in chunks
    constexpr size_t CHUNK_SIZE = 1024 * 1024; // 1MB
    std::string buffer;
    buffer.reserve(CHUNK_SIZE);

    for (const auto& line : data) {
        buffer += line + '\n';
        if (buffer.size() >= CHUNK_SIZE) {
            file.write(buffer.data(), buffer.size());
            buffer.clear();
        }
    }

    // Write remaining data
    if (!buffer.empty()) {
        file.write(buffer.data(), buffer.size());
    }
}
```

Key points for C++ implementation:
1. Use binary mode for better performance
2. Process data in chunks to manage memory usage
3. Implement external merge sort for sorting large files
4. Use buffered writing for better I/O performance
5. Consider memory mapping (mmap) for very large files:

```cpp
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>

void process_with_mmap(const char* filename) {
    int fd = open(filename, O_RDONLY);
    if (fd == -1) return;

    // Get file size
    struct stat sb;
    if (fstat(fd, &sb) == -1) {
        close(fd);
        return;
    }

    // Map file into memory
    char* addr = static_cast<char*>(mmap(nullptr, sb.st_size, 
                                       PROT_READ, MAP_PRIVATE, fd, 0));
    if (addr == MAP_FAILED) {
        close(fd);
        return;
    }

    // Process mapped memory here
    // ...

    // Cleanup
    munmap(addr, sb.st_size);
    close(fd);
}
```

For extremely large files, consider using memory-mapped files (mmap) as they can provide better performance by letting the OS handle memory management and caching.

Now for C++ parallelization, here are several approaches:

1. Using std::thread for basic parallelization:

```cpp
#include <thread>
#include <vector>
#include <mutex>

void parallel_process_file(const std::string& filename) {
    std::ifstream file(filename, std::ios::binary);
    std::vector<std::thread> threads;
    std::mutex file_mutex;
    const int num_threads = std::thread::hardware_concurrency();
    
    // Get file size
    file.seekg(0, std::ios::end);
    size_t file_size = file.tellg();
    file.seekg(0, std::ios::beg);
    
    size_t chunk_size = file_size / num_threads;
    
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back([&, i]() {
            std::vector<char> buffer(chunk_size);
            
            // Thread-safe file reading
            {
                std::lock_guard<std::mutex> lock(file_mutex);
                file.seekg(i * chunk_size);
                file.read(buffer.data(), chunk_size);
            }
            
            process_chunk(buffer);
        });
    }
    
    for (auto& thread : threads) {
        thread.join();
    }
}
```

2. Using OpenMP for simpler parallelization:

```cpp
#include <omp.h>

void parallel_process_vector(std::vector<std::string>& data) {
    #pragma omp parallel for
    for (size_t i = 0; i < data.size(); ++i) {
        process_item(data[i]);
    }
}

// Parallel merge sort
void parallel_merge_sort(std::vector<std::string>& data) {
    #pragma omp parallel
    {
        #pragma omp single
        {
            #pragma omp task
            merge_sort(data.begin(), data.begin() + data.size()/2);
            
            #pragma omp task
            merge_sort(data.begin() + data.size()/2, data.end());
        }
    }
    #pragma omp taskwait
    
    std::inplace_merge(data.begin(), data.begin() + data.size()/2, data.end());
}
```

3. Using std::async for task-based parallelism:

```cpp
#include <future>

void parallel_process_chunks(const std::string& filename) {
    const size_t CHUNK_SIZE = 1024 * 1024;  // 1MB chunks
    std::vector<std::future<void>> futures;
    
    std::ifstream file(filename, std::ios::binary);
    while (file) {
        auto chunk = std::make_shared<std::vector<char>>(CHUNK_SIZE);
        file.read(chunk->data(), CHUNK_SIZE);
        size_t bytes_read = file.gcount();
        
        if (bytes_read > 0) {
            futures.push_back(std::async(std::launch::async,
                [chunk, bytes_read]() {
                    process_chunk(chunk->data(), bytes_read);
                }
            ));
        }
    }
    
    // Wait for all tasks to complete
    for (auto& future : futures) {
        future.wait();
    }
}
```

4. Memory-mapped files with parallel processing:

```cpp
#include <sys/mman.h>

void parallel_mmap_process(const char* filename) {
    int fd = open(filename, O_RDONLY);
    if (fd == -1) return;
    
    struct stat sb;
    fstat(fd, &sb);
    
    char* addr = static_cast<char*>(mmap(nullptr, sb.st_size, 
                                       PROT_READ, MAP_PRIVATE, fd, 0));
    
    const int num_threads = std::thread::hardware_concurrency();
    size_t chunk_size = sb.st_size / num_threads;
    std::vector<std::thread> threads;
    
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back([=]() {
            char* chunk_start = addr + (i * chunk_size);
            size_t process_size = (i == num_threads - 1) ? 
                                 sb.st_size - (i * chunk_size) : chunk_size;
            process_chunk(chunk_start, process_size);
        });
    }
    
    for (auto& thread : threads) {
        thread.join();
    }
    
    munmap(addr, sb.st_size);
    close(fd);
}
```

5. Using thread pools for better resource management:

```cpp
class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;

public:
    ThreadPool(size_t threads) : stop(false) {
        for (size_t i = 0; i < threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queue_mutex);
                        condition.wait(lock,
                            [this] { return stop || !tasks.empty(); });
                        if (stop && tasks.empty()) return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
                }
            });
        }
    }

    template<class F>
    void enqueue(F&& f) {
        {
            std::unique_lock<std::mutex> lock(queue_mutex);
            tasks.emplace(std::forward<F>(f));
        }
        condition.notify_one();
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queue_mutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread& worker : workers) {
            worker.join();
        }
    }
};
```

Key considerations for parallelization:
1. Balance chunk sizes to avoid thread overhead
2. Handle thread synchronization carefully
3. Consider CPU cache effects
4. Watch for I/O bottlenecks
5. Use appropriate synchronization primitives
6. Consider using a thread pool for repeated tasks
7. Monitor system resources (CPU, memory, I/O) during parallel processing
