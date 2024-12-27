# Cpu_Parrilisation_Trains
Updated Trains parallelisation, using C++ and different parallelisation techniques to create a locking train system.

!!Check Master for Loadable Files!!


    //Skelpo
    //Modular and Resource Friendly Trains with Parralization

Libraries

    #include <iostream>//for input output operations
    #include <vector>//dynamic array functionality
    #include <thread>//enables multi-threading
    #include <mutex>//provides mutual exclusion prmitives
    #include <condition_variable>//enables synchronization via condtitions
    #include <memory>//supports dynamaic memory managment 
    #include <atomic>//ensures opersations are performend witout interuption

Colour controller 

    const char* ANSI_RESET = "\033[0m";
    const char* ANSI_RED = "\033[41m";
    const char* ANSI_GREEN = "\033[42m";
    const char* ANSI_BLUE = "\033[44m";
    const char* ANSI_YELLOW = "\033[43m";
 
 Inital member class
 
    class RailwaySystem {
    
    public:
        // Constructor for the RailwaySystem class, initializing the system's configuration.
        RailwaySystem(int numTracks, int trainsPerTrack, int trackLength) : totalLength(trackLength), segmentLength(10), shortStationLength(2), simulationActive(true) {segments.resize(numTracks); positions.resize(numTracks);

    //iterate over track  segments
        for (int i = 0; i < numTracks; ++i) { 

            segments[i].resize(totalLength);//resize each segment vector for each totalLength
            positions[i].resize(trainsPerTrack);//setup position tracker for each train based on the train on the track
            //initialize each segment along the track
            for (int j = 0; j < totalLength; ++j) {

                segments[i][j] = std::make_unique<Segment>();
            }//set up initial positions and mutexes for each train
            for (int k = 0; k < trainsPerTrack; ++k) {

                positions[i][k] = std::make_unique<std::atomic<int>>(k % 2 == 0 ? 0 : totalLength - 1);//setup position for each train; alternating between divisable by 2 trains and non divisable trains 
            }
        }
    }

    void startSimulation();//simulation functio
    ~RailwaySystem() {}//destructor

    private:
        struct Segment {//segment structure
            std::mutex mutex;
            std::condition_variable cond;
            bool occupied = false;
        };

    std::vector<std::vector<std::unique_ptr<Segment>>> segments;//track segmentation
    std::vector<std::vector<std::unique_ptr<std::atomic<int>>>> positions;
    std::mutex displayMutex;
    std::atomic<bool> simulationActive;
    const int segmentLength, shortStationLength, totalLength;//track information

    void moveTrain(int track, int train);
    void displayTracks();
    };

Run through simulation function

    void RailwaySystem::startSimulation() {

    std::vector<std::thread> threads;//creating vectoral threads

    for (size_t track = 0; track < segments.size(); ++track) {//creating track dimension

        for (size_t train = 0; train < positions[track].size(); ++train) {//creating trains threads
            threads.emplace_back(&RailwaySystem::moveTrain, this, track, train);//each threads infomration rearding train peramaters
        }
    }
  
    std::thread displayThread(&RailwaySystem::displayTracks, this);//creating display thread 

    for (auto& thread : threads) {
        thread.join();//joining train threads
    }

    simulationActive = false;
    displayThread.join();//joining display thread
    }
    
    void RailwaySystem::moveTrain(int track, int train) {

    while (simulationActive) {
        try {

        int pos = positions[track][train]->load();
        int direction = (train % 2 == 0) ? 1 : -1;  // even trains (Train A) forward, odd trains (Train B) backward
        int nextPos = (pos + direction + totalLength) % totalLength;  // wrap-around logic ensures correct boundary behavior

        int currentSegment = pos / (segmentLength + shortStationLength);//find current segment for train (Train A) and (Train B)
        int nextSegment = nextPos / (segmentLength + shortStationLength);

        // Calculate the segment index for the other train on the same track
        int otherTrainSegment = positions[track][(train + 1) % 2]->load() / (segmentLength + shortStationLength);

        // Check if Train B is trying to enter the same segment as Train A 
        if (train = train % 2) 
        if (otherTrainSegment == nextSegment) {
            // Lock trains B,D,F until the segment is clear
            std::unique_lock<std::mutex> lock(segments[track][nextSegment]->mutex);
            segments[track][nextSegment]->cond.wait(lock, [&] { return !segments[track][nextSegment]->occupied; });
        }
        

        // Release the current segment
        segments[track][currentSegment]->occupied = false;
        segments[track][currentSegment]->cond.notify_one();

        // Occupy the next segment
        segments[track][nextSegment]->occupied = true;

        // Update the train's position
        positions[track][train]->store(nextPos);

        // Sleep to simulate movement time
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
        }

        catch (const std::exception& e) {
            std::cerr << "Exception in moveTrain: " << e.what() << std::endl;
            break;  // Exit on exception or handle it as needed
        }
    }
    }
Display railway track

    void RailwaySystem::displayTracks() {
        // Loop until the simulation is active, continuously updating the console output
        while (simulationActive) {

        // Lock the display to prevent other threads from writing to the console simultaneously
        std::lock_guard<std::mutex> lock(displayMutex);
        // Clear the terminal screen to prepare for a fresh display of track data
        std::cout << "\x1B[2J\x1B[H";

        // Iterate through each track in the system
        for (size_t track = 0; track < segments.size(); ++track) {
            // Output the track number
            std::cout << "Track " << track + 1 << ":\n";

            // Iterate through each position on the track
            for (int pos = 0; pos < totalLength; ++pos) {

                // Calculate the index of the segment based on the current position
                int segmentIndex = pos / (segmentLength + shortStationLength);
                // Determine if the current position is a station
                bool isStation = (pos % (segmentLength + shortStationLength)) >= segmentLength;
                // Check if the segment at the current position is occupied
                bool occupied = segments[track][segmentIndex]->occupied;

                // Color the start/end of the track green
                if (pos == 0 || pos == totalLength - 1) {
                    std::cout << ANSI_GREEN;
                }
                // Color station segments blue
                else if (isStation) {
                    std::cout << ANSI_BLUE;
                }
                // Color occupied segments yellow
                else if (occupied) {
                    std::cout << ANSI_YELLOW;
                }

                // Check if there is a train at the current position
                bool trainHere = false;
                for (size_t train = 0; train < positions[track].size(); ++train) {
                    if (*(positions[track][train]) == pos) {
                        // Calculate and display the train label based on the track and train index
                        char trainLabel = 'A' + static_cast<char>(2 * track + (train % 2));
                        std::cout << trainLabel;
                        trainHere = true;
                        break;
                    }
                }

                // If no train is present, display an underscore
                if (!trainHere) std::cout << '_';
                // Reset the terminal color formatting
                std::cout << ANSI_RESET;
            }
            std::cout << "\n";
        }
        // Pause briefly before the next update to reduce flickering and CPU usage
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
    }

Run program

    int main() {
        RailwaySystem system(3, 6, 60);//run railway system and pass peramaters through concering railway and train systems
        system.startSimulation();//sun the simulation
        return 0;
    }
