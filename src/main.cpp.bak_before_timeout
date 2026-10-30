#include <array>
#include <cerrno>
#include <csignal>
#include <cstdint>
#include <cstdio>
#include <cstring>

#include <fcntl.h>
#include <poll.h>
#include <termios.h>
#include <unistd.h>

namespace
{
volatile std::sig_atomic_t g_running = 1;

constexpr const char *UART_DEVICE = "/dev/ttyAMA0";

constexpr std::size_t PACKET_SIZE = 151U;
constexpr std::size_t HEADER_SIZE = 10U;
constexpr std::size_t PAYLOAD_SIZE = 139U;
constexpr std::size_t CRC_OFFSET = 149U;

constexpr std::uint8_t MAGIC0 = 0x47U;
constexpr std::uint8_t MAGIC1 = 0x53U;
constexpr std::uint8_t VERSION = 0x01U;
constexpr std::uint8_t TYPE_STATUS = 0x01U;

constexpr std::uint16_t CRC16_INIT = 0xFFFFU;
constexpr std::uint16_t CRC16_POLYNOMIAL = 0xA001U;

void SignalHandler(int)
{
    g_running = 0;
}

bool ConfigureUart(int fd)
{
    struct termios tty {};

    if (tcgetattr(fd, &tty) != 0)
    {
        std::perror("tcgetattr");
        return false;
    }

    cfmakeraw(&tty);

    if (cfsetispeed(&tty, B115200) != 0 ||
        cfsetospeed(&tty, B115200) != 0)
    {
        std::perror("cfsetspeed");
        return false;
    }

    tty.c_cflag &= ~CSIZE;
    tty.c_cflag |= CS8;
    tty.c_cflag &= ~PARENB;
    tty.c_cflag &= ~CSTOPB;
    tty.c_cflag &= ~CRTSCTS;
    tty.c_cflag |= CLOCAL | CREAD;

    tty.c_cc[VMIN] = 1;
    tty.c_cc[VTIME] = 0;

    if (tcsetattr(fd, TCSANOW, &tty) != 0)
    {
        std::perror("tcsetattr");
        return false;
    }

    return true;
}

std::uint16_t ReadLE16(const std::uint8_t *data)
{
    return static_cast<std::uint16_t>(
        static_cast<std::uint16_t>(data[0]) |
        (static_cast<std::uint16_t>(data[1]) << 8U)
    );
}

std::uint32_t ReadLE32(const std::uint8_t *data)
{
    return static_cast<std::uint32_t>(data[0]) |
           (static_cast<std::uint32_t>(data[1]) << 8U) |
           (static_cast<std::uint32_t>(data[2]) << 16U) |
           (static_cast<std::uint32_t>(data[3]) << 24U);
}

std::uint16_t CalculateCRC16(
    const std::uint8_t *data,
    std::size_t length)
{
    std::uint16_t crc = CRC16_INIT;

    for (std::size_t i = 0; i < length; ++i)
    {
        crc ^= data[i];

        for (unsigned int bit = 0; bit < 8U; ++bit)
        {
            if ((crc & 1U) != 0U)
            {
                crc = static_cast<std::uint16_t>(
                    (crc >> 1U) ^ CRC16_POLYNOMIAL
                );
            }
            else
            {
                crc >>= 1U;
            }
        }
    }

    return crc;
}

struct Statistics
{
    std::uint64_t total_bytes = 0U;
    std::uint64_t valid_packets = 0U;
    std::uint64_t invalid_headers = 0U;
    std::uint64_t crc_errors = 0U;
    std::uint64_t payload_errors = 0U;
};

bool ValidatePacket(
    const std::uint8_t *packet,
    Statistics &stats)
{
    if (packet[0] != MAGIC0 ||
        packet[1] != MAGIC1 ||
        packet[2] != VERSION ||
        packet[3] != TYPE_STATUS ||
        ReadLE16(&packet[8]) != PAYLOAD_SIZE)
    {
        ++stats.invalid_headers;
        return false;
    }

    if (packet[10] != '>' ||
        packet[CRC_OFFSET - 2U] != '\r' ||
        packet[CRC_OFFSET - 1U] != '\n')
    {
        ++stats.payload_errors;
        return false;
    }

    const std::uint16_t calculated_crc =
        CalculateCRC16(packet, CRC_OFFSET);

    const std::uint16_t received_crc =
        ReadLE16(&packet[CRC_OFFSET]);

    if (calculated_crc != received_crc)
    {
        ++stats.crc_errors;
        return false;
    }

    return true;
}

void PrintPacket(
    const std::uint8_t *packet,
    const Statistics &stats)
{
    const std::uint32_t sequence = ReadLE32(&packet[4]);
    const std::uint16_t payload_length = ReadLE16(&packet[8]);
    const std::uint16_t received_crc = ReadLE16(&packet[CRC_OFFSET]);

    std::printf(
        "VALID packet #%llu: seq=%u, payload=%u, "
        "CRC=0x%04X, total_bytes=%llu\n",
        static_cast<unsigned long long>(stats.valid_packets),
        static_cast<unsigned int>(sequence),
        static_cast<unsigned int>(payload_length),
        static_cast<unsigned int>(received_crc),
        static_cast<unsigned long long>(stats.total_bytes)
    );

    // Print a short payload preview for troubleshooting.
    std::printf("  Payload preview: ");

    for (std::size_t i = 0; i < 17U; ++i)
    {
        const unsigned char ch = packet[10U + i];

        if (ch >= 0x20U && ch <= 0x7EU)
        {
            std::putchar(ch);
        }
        else
        {
            std::printf("\\x%02X", ch);
        }
    }

    std::printf("...\n");
    std::fflush(stdout);
}

void ProcessReceivedData(
    std::array<std::uint8_t, 4096U> &buffer,
    std::size_t &buffer_size,
    const std::uint8_t *data,
    std::size_t length,
    Statistics &stats)
{
    for (std::size_t i = 0; i < length; ++i)
    {
        if (buffer_size >= buffer.size())
        {
            // Defensive recovery: discard buffered data if full.
            buffer_size = 0U;
        }

        buffer[buffer_size++] = data[i];
        ++stats.total_bytes;

        while (buffer_size >= 2U)
        {
            // Find the GS magic header.
            std::size_t magic_pos = 0U;

            while (magic_pos + 1U < buffer_size)
            {
                if (buffer[magic_pos] == MAGIC0 &&
                    buffer[magic_pos + 1U] == MAGIC1)
                {
                    break;
                }

                ++magic_pos;
            }

            if (magic_pos + 1U >= buffer_size)
            {
                const bool keep_last_byte =
                    buffer[buffer_size - 1U] == MAGIC0;

                if (keep_last_byte)
                {
                    buffer[0] = MAGIC0;
                    buffer_size = 1U;
                }
                else
                {
                    buffer_size = 0U;
                }

                break;
            }

            if (magic_pos > 0U)
            {
                std::memmove(
                    buffer.data(),
                    buffer.data() + magic_pos,
                    buffer_size - magic_pos
                );

                buffer_size -= magic_pos;
            }

            if (buffer_size < PACKET_SIZE)
            {
                break;
            }

            if (!ValidatePacket(buffer.data(), stats))
            {
                // Move one byte forward and search for the next header.
                std::memmove(
                    buffer.data(),
                    buffer.data() + 1U,
                    buffer_size - 1U
                );

                --buffer_size;
                continue;
            }

            ++stats.valid_packets;
            PrintPacket(buffer.data(), stats);

            const std::size_t remaining =
                buffer_size - PACKET_SIZE;

            if (remaining > 0U)
            {
                std::memmove(
                    buffer.data(),
                    buffer.data() + PACKET_SIZE,
                    remaining
                );
            }

            buffer_size = remaining;
        }
    }
}
}

int main()
{
    std::signal(SIGINT, SignalHandler);
    std::signal(SIGTERM, SignalHandler);

    const int uart_fd =
        open(UART_DEVICE, O_RDONLY | O_NOCTTY | O_NONBLOCK);

    if (uart_fd < 0)
    {
        std::perror("open UART");
        return 1;
    }

    if (!ConfigureUart(uart_fd))
    {
        close(uart_fd);
        return 1;
    }

    std::printf("UART device : %s\n", UART_DEVICE);
    std::printf("UART format : 115200 baud, 8N1\n");
    std::printf("Packet size : %zu bytes\n", PACKET_SIZE);
    std::printf("Waiting for Status Stream packets... Press Ctrl+C to exit.\n");
    std::fflush(stdout);

    struct pollfd pfd {};
    pfd.fd = uart_fd;
    pfd.events = POLLIN;

    std::array<std::uint8_t, 4096U> packet_buffer {};
    std::size_t packet_buffer_size = 0U;

    std::uint8_t rx_buffer[256U] {};
    Statistics stats {};

    while (g_running)
    {
        const int poll_result = poll(&pfd, 1, 500);

        if (poll_result < 0)
        {
            if (errno == EINTR)
            {
                continue;
            }

            std::perror("poll");
            break;
        }

        if (poll_result == 0)
        {
            continue;
        }

        if (pfd.revents & POLLNVAL)
        {
            std::fprintf(stderr, "UART file descriptor is invalid.\n");
            break;
        }

        if (pfd.revents & POLLIN)
        {
            const ssize_t bytes_read =
                read(uart_fd, rx_buffer, sizeof(rx_buffer));

            if (bytes_read > 0)
            {
                ProcessReceivedData(
                    packet_buffer,
                    packet_buffer_size,
                    rx_buffer,
                    static_cast<std::size_t>(bytes_read),
                    stats
                );
            }
            else if (bytes_read < 0 &&
                     errno != EAGAIN &&
                     errno != EWOULDBLOCK &&
                     errno != EINTR)
            {
                std::perror("read UART");
                break;
            }
        }

        if (pfd.revents & (POLLERR | POLLHUP))
        {
            std::fprintf(stderr, "UART reported an error or hangup.\n");
            break;
        }
    }

    std::printf(
        "\nUART receiver stopped.\n"
        "Total bytes: %llu\n"
        "Valid packets: %llu\n"
        "Invalid headers: %llu\n"
        "Payload errors: %llu\n"
        "CRC errors: %llu\n",
        static_cast<unsigned long long>(stats.total_bytes),
        static_cast<unsigned long long>(stats.valid_packets),
        static_cast<unsigned long long>(stats.invalid_headers),
        static_cast<unsigned long long>(stats.payload_errors),
        static_cast<unsigned long long>(stats.crc_errors)
    );

    close(uart_fd);
    return 0;
}