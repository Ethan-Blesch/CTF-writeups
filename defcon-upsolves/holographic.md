# Holographic
This challenge generates a deck of cards seeded from two numbers, one of which changes and one of which is a constant. You then have to guess the shuffled card order, and importantly, are shown the correct answer after guessing wrong.<br></br>
By iterating through possible seeds for the second value (which can range from 0 to 0xFFFFFF), we can plug in the given seed & the correct shuffled deck, and then brute-force the secret number.
```cpp
int main()                                    
{  
        ChallRng::result_type knownSeed;
        ChallRng::result_type testSeed = 1337;
        Deck knownDeck{};
        std::cout << "Enter the deck: \n";                                                            
        std::cin >> knownDeck;
        std::cout << "Enter the seed: \n";        
        std::cin >> knownSeed;
        std::cout << knownSeed << "\n";
        while(true){
                std::cout << "Trying seed " << testSeed << "\n"; 
                ChallRng rng(testSeed, knownSeed);
                Deck testDeck(rng);
                std::cout << testDeck;
                if(testDeck == knownDeck){                                                            
                        std::cout << "FOUND\n";
                        std::cout << testSeed;
                        break;
                }
                break;
                testSeed++;

        }
}
```
Then, we can connect again, and use the seed we found and the seed we're given to generate the correct answer and be given the flag. 
