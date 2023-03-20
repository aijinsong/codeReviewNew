package cn.loopcloud.file.util;

import cn.loopcloud.exception.biz.BizValidationException;
import cn.loopcloud.sharing.log.Logger;
import cn.loopcloud.sharing.log.LoggerFactory;
import cn.loopcloud.sharing.util.Strings;
import net.coobird.thumbnailator.Thumbnails;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.rendering.PDFRenderer;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/**
 * 文件压缩工具
 *
 * @author zhangning
 * @since 2021/3/27 15:06
 */
public class FileThumbUtils {

    private Logger logger = LoggerFactory.getLogger(this.getClass());

    public byte[] thumbFileByParam(File workFile) throws IOException {
        return thumbFileByParam(workFile, 30f);
    }

    public byte[] thumbFileByParam(File workFile, float pdfDpi) throws IOException {
        if (!workFile.exists() || !workFile.isFile()) {
            throw new BizValidationException("不存在的文件:" + workFile.getAbsolutePath());
        }
        String name = workFile.getName();
        if (!name.endsWith(".pdf") && !name.endsWith(".jpg") && !name.endsWith(".jpeg")) {
            logger.error("非pdf、jpg、jpeg文件:{}", workFile.getAbsolutePath());
            throw new BizValidationException(Strings.format("非pdf、jpg、jpeg文件:{}", workFile.getAbsolutePath()));
        }
        logger.info("开始压缩文件 {},{}", workFile.getAbsolutePath(), pdfDpi);
        if (name.endsWith(".pdf")) {
            // 由于imageMagic生成缩略图不符合标准, 这里统一先使用pdfbox生成
            // pdfbox导致内存爆掉的问题后续跟踪
            return thumbPdf(workFile, pdfDpi);
//            try {
//                return Im4JavaUtils.getInstance().compressImage(workFile, 150);
//            } catch (Throwable e) {
//                logger.error("imageMagick压缩文件失败:" + e.getMessage());
//                return thumbPdf(workFile, pdfDpi);
//            }
        } else {
            try {
                return Im4JavaUtils.getInstance().compressImage(workFile, 150);
            } catch (Throwable e) {
                logger.error("imageMagick压缩文件失败:" + e.getMessage());
                return thumbJpg(workFile);
            }
        }
    }

    protected byte[] thumbJpg(File workFile) throws IOException {
        try (ByteArrayOutputStream os = new ByteArrayOutputStream();) {
            long l = System.currentTimeMillis();
            logger.info("开始压缩文件 大小：{}", workFile.length());
            Thumbnails.of(workFile).scale(1).outputQuality(0.1f)
                    .toOutputStream(os);
            logger.info("结束压缩文件 所用时长{}", System.currentTimeMillis() - l);
            return os.toByteArray();
        } catch (Exception e) {
            logger.error("压缩文件异常", e);
            throw e;
        }
    }

    protected byte[] thumbPdf(File workFile, float pdfDpi) throws IOException {
        logger.info("开始pdfbox生成压缩图片大小：{}, {}",
                workFile.getAbsolutePath(), workFile.length());
        System.setProperty("sun.java2d.cmm", "sun.java2d.cmm.kcms.KcmsServiceProvider");
        try (PDDocument doc = PDDocument.load(workFile);
             ByteArrayOutputStream os = new ByteArrayOutputStream();) {
            PDFRenderer renderer = new PDFRenderer(doc);
            int pageCount = doc.getNumberOfPages();
            if (pageCount <= 0) {
                throw new BizValidationException("文件数据为空");
            }
            os.reset();
            long l = System.currentTimeMillis();
//                    float dpi = param.getBooleanValue("isOrder") ? 100 : 30;
            BufferedImage image = renderer.renderImageWithDPI(0, pdfDpi);
            ImageIO.write(image, "JPG", os);
            logger.info("结束压缩文件 所用时长{}", System.currentTimeMillis() - l);
            return os.toByteArray();
        } catch (Exception e) {
            logger.error("pdfbox压缩文件异常: {}", workFile.getAbsolutePath(), e);
            throw e;
        }
    }

    public byte[] thumbPdfOddPage(File workFile, float pdfDpi) throws IOException {
        if (!workFile.exists() || !workFile.isFile()) {
            throw new BizValidationException("不存在的文件:" + workFile.getAbsolutePath());
        }
        String name = workFile.getName();
        if (!name.endsWith(".pdf")) {
            logger.error("非pdf文件:{}", workFile.getAbsolutePath());
            throw new BizValidationException(Strings.format("非pdf文件:{}", workFile.getAbsolutePath()));
        }

        try {
            return Im4JavaUtils.getInstance().compressImage(workFile, 150, 50d, true);
        } catch (Throwable throwable) {
            logger.error("imageMagick压缩文件失败:" + throwable.getMessage());
            System.setProperty("sun.java2d.cmm", "sun.java2d.cmm.kcms.KcmsServiceProvider");
            try (PDDocument doc = PDDocument.load(workFile);) {
                PDFRenderer renderer = new PDFRenderer(doc);
                int pageCount = doc.getNumberOfPages();
                if (pageCount <= 0) {
                    throw new BizValidationException("文件数据为空");
                }
                try (ByteArrayOutputStream os = new ByteArrayOutputStream();) {
                    long l = System.currentTimeMillis();
                    logger.info("开始压缩文件 页数{} 大小：{}", pageCount, workFile.length());
                    List<BufferedImage> images = new ArrayList<>();
                    // 所有页width合计
                    int totalWidth = 0;
                    int maxHeight = 0;
                    for (int i = 0; i < pageCount + 1 >> 1; i++) {
                        BufferedImage bim = renderer.renderImageWithDPI(i << 1, pdfDpi);
                        totalWidth += bim.getWidth();
                        maxHeight = Math.max(maxHeight, bim.getHeight());
                        images.add(bim);
                    }
                    // 合并图片
                    BufferedImage pdfImage = new BufferedImage(totalWidth, maxHeight, BufferedImage.TYPE_INT_RGB);
                    int y = 0;
                    for (BufferedImage bim : images) {
                        // 将每页pdf画到总的pdfImage上,x坐标=0，y坐标=之前所有页的高度和，属于向下偏移
                        pdfImage.getGraphics().drawImage(bim, y, 0, null);
                        y += bim.getWidth();
                    }
                    ImageIO.write(pdfImage, "JPG", os);
                    logger.info("结束压缩文件 所用时长{}", System.currentTimeMillis() - l);
                    return os.toByteArray();
                }
            } catch (Exception e) {
                logger.error("压缩文件异常", e);
                throw e;
            }
        }
    }
}
