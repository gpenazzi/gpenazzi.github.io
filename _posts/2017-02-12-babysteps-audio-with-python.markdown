---
layout: post
title:  "Python audio babysteps"
date:   2017-02-12
categories: [python]
tags: [python, audio, signalprocessing, pyaudio]
---

If you like coding and you have some interest in audio synthesis, there is a good chance that you
tried to play around yourself. As you can guess, this turns out to be super fast and easy in python.
I want to acknowledge [Zach Denton's blog](https://zach.se/generate-audio-with-python/), his
post inspired me to play around a bit as well and introduced me to the concept of binaural beat.

What I want to do is being able to generate arbitrary signals with arbitrary duration and play
them directly from python, without exporting them. Eventually I would like to get to the point
where I can try to generate some noise pattern running indefinitely.

The first step is getting some sound out. I am going to use
[pyAudio](https://people.csail.mit.edu/hubert/pyaudio/), a library providing Python bindings for
[PortAudio](www.portaudio.com). pyAudio can be fed with data supporting the stream interface.
Numpy arrays can be exported to stream interface and will make a terrific job for the signal
generatio part, therefore let's start by playing a numpy array. I import the modules and define
a global bitrate.

{% highlight python %}
import numpy
import pyaudio

BITRATE = 44100
{% endhighlight %}


Then I define a function which generates a sine function with given frequency and duration.

{% highlight python %}
def sinOsc(frequency, duration, amplitude=1.0):
    nframes = duration * BITRATE
    frames = numpy.arange(nframes)
    frame_frequency = frequency / BITRATE
    val = amplitude * numpy.sin(frames * frame_frequency * 2.0 * numpy.pi)
    return val.astype(dtype=numpy.float32)
{% endhighlight %}

Note that we have to export as float32. pyAudio supports only some specific formats, including
integers and float32 but not double64 (the numpy default). It is quite convenient to work in double
precision and forget about the format till the end. It may be more efficient to work directly with
integers, but it's not going to be really a problem and I want to keep the code as simple as
possible. Now that we have a signal, playing it is a piece of cake.

{% highlight python %}
pa = pyaudio.PyAudio()

stream = pa.open(format=pyaudio.paFloat32,
                channels=1,
                rate=BITRATE,
                output=True)
# duration in seconds and frequency in Hertz
signal = sinOsc(frequency=440.0, duration=2.0)
stream.write(signal.tostring())

stream.stop_stream()
stream.close()
pa.terminate()
{% endhighlight %}

What about stereo sounds? Stereo signals are interleaved, i.e. they alternate left and right
samples. It is quite easy to interleave two numpy arrays. The sin generator does not need to be
modified, we can generate two signals, interleave them and play them out.

{% highlight python %}
pa = pyaudio.PyAudio()
stream = pa.open(format=pyaudio.paFloat32,
                channels=2,
                rate=BITRATE,
                output=True)

signal = sinOsc(frequency=440.0, duration=2.0)
# Stereo interleave
stereo_signal = numpy.ravel(numpy.column_stack((signal,signal)))

stream.write(stereo_signal.tostring())
stream.stop_stream()
stream.close()
pa.terminate()
{% endhighlight %}

It will sound the same as the mono version, as it should. But we can interleave two different
signals, for example oscillators with different frequency (binaural beat).

{% highlight python %}
pa = pyaudio.PyAudio()
stream = pa.open(format=pyaudio.paFloat32,
               channels=2,
               rate=BITRATE,
               output=True)

signal_left = sinOsc(frequency=440.0, duration=5.0)
signal_right = sinOsc(frequency=450.0, duration=5.0)
# Stereo interleave
stereo_signal = numpy.ravel(numpy.column_stack((signal_left,signal_right)))

stream.write(stereo_signal.tostring())
stream.stop_stream()
stream.close()
pa.terminate()
{% endhighlight %}

This approach for the signal generation is very simple, but inefficient as well. Whenever
we want to generate a signal, we have to build the whole numpy array. In general this is not
really what we want to do, if want a very long oscillator sound we could rather generate small
chunk of data and feed them to pyAudio indefinitely. A good starting point is to
reimplement the same oscillator as an iterator which returns chunk of data till an arbitrary
stop condition is set. Here below is a basic implementation. At first we can use the iterator to
generate the same array we had befor, and play it.

{% highlight python %}
import pyaudio
import numpy

BITRATE = 44100
BUFFERSIZE = 1024

class SinOsc(object):

    def __init__(self, frequency, amplitude=1.0):
        self._t = 0
        self._stop = False
        self._amplitude = amplitude
        self._frame_frequency = frequency / BITRATE
        self._frames = numpy.arange(BUFFERSIZE)
        self._chunk = numpy.zeros(BUFFERSIZE, dtype=numpy.float32)

    def _reset(self):
        self._t = 0
        self._stop = False
        self._chunk = numpy.zeros(BUFFERSIZE, dtype=numpy.float32)

    def __iter__(self):
        return self

    def __len__(self):
        return 1

    def next(self):
        if self._stop:
            raise StopIteration
        else:
            frames = self._frames + self._t
            chunk = self._amplitude * numpy.sin(frames * self._frame_frequency *
                    2.0 * numpy.pi)
            self._chunk = chunk.astype(numpy.float32)
            self._t += BUFFERSIZE
            return self._chunk

    def generateArray(self, duration):
        sequence = []
        for chunk in self.__iter__():
            sequence.append(chunk)
            if self._t / BITRATE > duration:
                self._stop = True
        self._reset()
        return numpy.concatenate(sequence)

osc = SinOsc(440.0)

pa = pyaudio.PyAudio()
stream = pa.open(format=pyaudio.paFloat32,
                channels=1,
                rate=BITRATE,
                output=True)

stream.write(osc.generateArray(duration=1.00).tostring())
stream.stop_stream()
stream.close()
pa.terminate()
{% endhighlight %}

The real body of the class is in the next() function, which provide the value returned by the
iterator. Once we have the iterator, have an infinite play is a piece of cake. We can
feed pyAudio chunk by chunk with a loop. The class definition is the same, but we call it in
this way

{% highlight python %}
osc = SinOsc(440.0)

pa = pyaudio.PyAudio()
stream = pa.open(format=pyaudio.paFloat32,
                 channels=1,
                 rate=BITRATE,
                 output=True)

try:
    for data in osc:
        stream.write(data.tostring())
except KeyboardInterrupt:
    stream.stop_stream()
    stream.close()
    pa.terminate()
    print('Audio terminated correctly')
{% endhighlight %}

This plays the sample indefinitely till we do not send an interrupt signal (CTRL-C on linux, for
  example).
