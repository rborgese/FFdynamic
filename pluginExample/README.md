## Write plugin - run your code easily with FFdynamic
---------

### Dehaze Plugin

Suppose we have developed a dehaze algorightm with openCV dependency, and we would like to use it in transcoding pipeline, namely, run it after video decode, encode the dehazed stream and then publish it. So the pattern may like this:

```
Demux |-> Audio Decode  |-> Audio Encode  ------------------------------> |
                                                                          |
      |-> Video Decode  |-> Dehaze Filter -> | Scale1 -> Video Encode1 -> | -> Muxer1
                                             | Scale2 -> Video Encode2 -> | -> Muxer2
                                             | Scale3 -> Video Encode3 -> | -> Muxer3
```

In FFmpeg, we would make dehaze as a filter, do according coding and makefile changing, then re-compile the FFmpeg.
In FFdynamic, we could write dehaze as a plugin and then link FFdynamic library without modify existing code.

#### Dehaze Plugin - FFdynamic implementation register and coding

There are two things needed:
1. coding dehaze implementation (inherits from FFdynamic's DavImpl class and implement according interface)
2. register this implementation

For **coding** part, it is just a make dehaze fit the data flow requirement of FFdynamic, please refer to the source.

For **register**, we have two ways, register it to an existing component category as an implementation or create an new component category and register it as implementation to the newly created category.
* register to existing one

``` c++
DavImplRegister g_dehazeReg(DavWaveClassVideoFilter(), vector<string>({"pluginDehazor"}),
                            [](const DavWaveOption & options) -> unique_ptr<DavImpl> {
                                     unique_ptr<PluginDehazor> p(new PluginDehazor(options));
                                     return p;
                            });
```
Here, we register 'PluginDehazor' as an implementation of component 'VideoFilter';

* create new category and do register

``` c++
/* create dehaze component category */
struct DavWaveClassDehaze : public DavWaveClassCategory {
    DavWaveClassDehaze () : 
        DavWaveClassCategory(type_index(typeid(*this)), type_index(typeid(std::string)), "Dehaze") {}
};

/* Then register PluginDehazor as default dehazor (by provides 'auto') */
DavImplRegister g_dehazeReg(DavWaveClassVideoDehaze(), vector<string>({"auto", "dehaze"}),
                            [](const DavWaveOption & options) -> unique_ptr<DavImpl> {
                                unique_ptr<PluginDehazor> p(new PluginDehazor(options));
                                return p;
                            });

```

#### Define an static option (used when create the component)

For options passing, there are two ways.  
* Define derived clsss from DavOption:

``` c++
struct DavOptionDehazeFogFactor : public DavOption {
    DavOptionDehazeFogFactor() :
        DavOption(type_index(typeid(*this)), type_index(typeid(double)), "DehazeFogFactor") {}
};

// Then set/get it like this:

DavWaveOption videoDehazeOption((DavWaveClassDehaze()));
videoDehazeOption.setDouble(DavOptionDehazeFogFactor(), 0.94);

double fogFactor = 0.94;
videoDehazeOption.getDouble(DavOptionDehazeFogFactor(), fogFactor);

```

* Use 'AVDictionary' (just the FFmpeg's way, flexiable but not type checked)

``` c++
       videoDehazeOption.setDouble("FogFactor", 0.94);
       // then in dehaze component, we could:
       double fogFactor;
       int ret = options.getDouble("FogFactor", fogFactor);
       if (ret < 0)
       .......
```

So, the rule of thumb:
* defines derived DavOption class for component level options; like: DavOptionInputUrl for all demuxer
* use AVDictionary for implementation level options. 

#### Register an dynamic event

We could register dynamic event to change the paramters settings during running.  
This is an example that we register an run time event that would change Dehaze's FogFacotr during process.

``` c++
    std::function<int (const FogFactorChangeEvent &)> f =
        [this] (const FogFactorChangeEvent & e) {return processFogFactorUpdate(e);};
    m_implEvent.registerEvent(f);
```

#### Test dehaze with visualization

For instance, if we would like to know how good is a dehaze filter visually (in compare to original one), we could use the following pattern to mix dehazed and original video frame in one screen by following composition.

```
Demux |-> Audio Decode  |-> Audio Encode  ---------------------------> |
                                                                       | -> Muxer
      |-> Video Decode  |-> Dehaze Filter -> | Video Mix and Encode -> |
                        |->----------------> |
```

The result is this (the dehaze effect is not good, but it is not the point).
![dehazed mix image](../asset/dehze.gif)


You can refer to the source file [here](ffdynaDehazor.cpp). (NOTE: the dehazed algorightm itself is from github, but miss the link).

#### Run the dehazed example

``` 
    mkdir build && cmake ../ && make
    ./testDehazor theInputFile
```
