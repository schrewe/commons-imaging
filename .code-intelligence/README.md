# Fuzzing the Apache Commons Imaging Java Image Library

## What is Commons Imaging?

Apache Commons Imaging, is a library 
that reads and writes a variety of image formats, including fast parsing of 
image info (size, color space, ICC profile, etc.) and metadata.

## The problem

Even in a memory-safe language such as Java, 
potentially harmful bugs like uncaught
exceptions or denial of service problems are an issue.
Writing a parser is particularly hard, because the developer needs to think
about all edge cases and invalid inputs in addition to handling correct inputs.

## The solution

With fuzz testing, finding edge cases is easy and automated, because the fuzzer
holds no assumptions about the format of the input. Commons Imaging allows you to process
arbitrary data using a collection of parsers for different image formats including jpeg, pgn, bnp and gif.

Fuzzing is a dynamic code analysis technique that supplies pseudo-random inputs
to a software-under-test (SUT), derives new inputs from the behaviour of the
program (i.e. how inputs are processed), and monitors the SUT for bugs.

## The setup

### Fuzzing where raw data is handled

Fuzzing is most efficient where raw data is parsed, because in this case no
assumptions can be made about the format of the input. Commons Imaging allows you to pass
arbitrary data to an encode function `getBufferedImage` and populates a
`BufferedImage` from that.

The most universal example of this type of fuzz test can be found in
[`.code-intelligence/fuzz_targets/AllImageParser.java`](https://github.com/ci-fuzz/commons-imaging/blob/master/.code-intelligence/fuzz_targets/AllImageParser.java).
Let me walk you through the heart of the fuzz test:


```Java
public class AllImageParser {
// 1. The function fuzzerTestOneInput will be repeatedly executed using data generated by the fuzzer as input
	public static void fuzzerTestOneInput(FuzzedDataProvider data) {

		final BufferedImage image;
		try{
// 2. Consume the fuzzing bytes and try to interpret them as image by automatically detecting the format inernally.
			byte[] imagebytes = data.consumeBytes(data.consumeInt(1,Integer.MAX_VALUE));
			image = Imaging.getBufferedImage(imagebytes);
			final IImageMetadata metadata = Imaging.getMetadata(imagebytes);
			final ICC_Profile iccProfile = Imaging.getICCProfile(imagebytes);
			final ImageInfo imageInfo = Imaging.getImageInfo(imagebytes);

		} catch(ImageReadException | IOException e){
			return;
		}
		final ImageFormat format = ImageFormats.values()[data.consumeInt(0,14)];
		final Map<String, Object> params = new HashMap<>();

		try{
// 3. Try to convert the image into another format that is selected based on the fuzzer input
        	Imaging.writeImageToBytes(image, format, params);
		} catch(ImageWriteException | IOException e){
			return;
		}
	}
}
```

If you haven't done already, you can now explore what the fuzzer found when
running this fuzz test.

### A note regarding corpus data (and why there are more fuzz tests to explore)

For each fuzz test that we write, a corpus of interesting inputs is built up.
Over time, the fuzzer will add more and more inputs to this corpus, based
coverage metrics such as newly-covered lines, statements or even values in an
expression.

The rule of thumb for a good fuzz test is that the format of the inputs should
be roughly the same. Therefore, it is sensible to split up the big fuzz test for
all barcode types into individual fuzz tests.

### Fuzzing in CI/CD

CI Fuzz allows you to configure your pipeline to automatically trigger the run of fuzz tests.
Most of the fuzzing runs that you can inspect here were triggered automatically (e.g. by pull or merge request on the GitHub project).
As you can see in this [`pull request`](https://github.com/ci-fuzz/commons-imaging/pull/12) the fuzzing results are automatically commented by the github-action and developers
can consume the results by clicking on "View Finding" which will lead them directly to the bug description with all the details
that CI Fuzz provides (input that caused the bug, stack trace, bug location).
With this configuration comes the hidden strength of fuzzing into play:  
Fuzzing is not like a penetration test where your application will be tested one time only.
Once you have configured your fuzz test it can help you for the whole rest of your developing cycle.
By running your fuzz test each time when some changes where made to the source code you can quickly check for
regressions and also quickly identify new introduced bugs that would otherwise turn up possibly months 
later during a penetration test or (even worse) in production. This can help to significantly reduce the bug ramp down phase of any project.

While these demo projects are configured to trigger fuzzing runs on merge or pull requests
there are many other configuration options for integrating fuzz testing into your CI/CD pipeline
for example you could also configure your CI/CD to run nightly fuzz tests.
